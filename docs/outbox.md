# Outbox

Use the outbox pattern when one workflow must:

1. persist business state in SQL, and
2. publish an integration message to an external broker.

Without an outbox, these are two separate writes and can diverge under failures.

## Table of Contents

1. [What is the Outbox Pattern?](#what-is-the-outbox-pattern)
2. [Why use outbox with event-sourced systems?](#why-use-outbox-with-event-sourced-systems)
3. [Watermill as an outbox engine](#watermill-as-an-outbox-engine)
4. [Transactional publish flow (SQL)](#transactional-publish-flow-sql)
5. [Forwarding to RabbitMQ (example)](#forwarding-to-rabbitmq-example)
6. [Delivery guarantees and ordering](#delivery-guarantees-and-ordering)
7. [Poison queue: why it is necessary](#poison-queue-why-it-is-necessary)
8. [Operational checklist](#operational-checklist)
9. [Troubleshooting](#troubleshooting)

## What is the Outbox Pattern?

The outbox pattern stages outbound messages in the same SQL transaction as your business write, then forwards those staged messages to the broker asynchronously.

This closes the classic dual-write gap:

- **Write succeeds, publish fails** -> state changed but no event delivered.
- **Publish succeeds, write fails** -> event delivered for state that does not exist.

The outbox pattern is the practical alternative to 2PC for most systems.

## Why use outbox with event-sourced systems?

Event sourcing and outbox solve adjacent but different boundaries:

- **Event store**: reliable domain persistence and replay.
- **Outbox**: reliable delivery boundary from SQL to external transports.

Even if event storage is already transactional, integration delivery still needs explicit handling. Outbox keeps persistence concerns and transport concerns separated while preserving correctness.

## Watermill as an outbox engine

Watermill gives you the two core building blocks:

- **SQL Publisher**: can publish within an active SQL transaction (`*sql.Tx`).
- **Forwarder**: subscribes to SQL-staged messages and republishes them to another broker (RabbitMQ in this guide).

Reference flow:

```text
App command/handler
  -> SQL transaction begins
  -> business state persisted
  -> integration message staged to SQL (forwarder topic envelope)
  -> transaction commit

Forwarder worker (separate process)
  -> reads staged messages from SQL
  -> republishes to RabbitMQ topic/queue
  -> acknowledges SQL offset
```

## Transactional publish flow (SQL)

Recommended write-time sequence:

1. Begin SQL transaction.
2. Append domain event(s) with `store.Append(...)`.
3. Create Watermill SQL publisher bound to that `*sql.Tx`.
4. Decorate publisher with `forwarder.NewPublisher(...)`.
5. Publish integration message(s).
6. Commit transaction.
7. On error, rollback (state and staged messages are both reverted).

### Transaction-bound append + publish snippet (Go)

Assume the standard setup from Getting Started already exists:

- `db *sql.DB`
- `store := postgres.NewStore(postgres.DefaultStoreConfig())`

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
	return err
}
defer tx.Rollback()

domainPayload, err := json.Marshal(map[string]string{
	"email": email,
})
if err != nil {
	return err
}

events := []es.Event{
	{
		BoundedContext: "Identity",
		AggregateType:  "User",
		AggregateID:    userID,
		EventID:        uuid.New(),
		EventType:      "UserRegistered",
		EventVersion:   1,
		Payload:        domainPayload,
		Metadata:       []byte(`{}`),
		CreatedAt:      time.Now(),
	},
}

// 1) Persist domain events with pupsourcing event store.
if _, err := store.Append(ctx, tx, es.NoStream(), events); err != nil {
	return err
}

sqlPublisher, err := wmsql.NewPublisher(
	wmsql.TxFromStdSQL(tx),
	wmsql.PublisherConfig{
		SchemaAdapter: wmsql.DefaultPostgreSQLSchema{},
	},
	watermill.NopLogger{},
)
if err != nil {
	return err
}
defer sqlPublisher.Close()

outboxPublisher := forwarder.NewPublisher(sqlPublisher, forwarder.PublisherConfig{
	ForwarderTopic: "outbox.forwarder",
})

integrationPayload, err := json.Marshal(map[string]string{
	"user_id": userID,
	"email":   email,
})
if err != nil {
	return err
}

msg := message.NewMessage(watermill.NewUUID(), integrationPayload)
msg.Metadata.Set("event_type", "UserRegistered")
msg.Metadata.Set("aggregate_id", userID)

// 2) Stage integration message in SQL outbox, still in the same tx.
if err := outboxPublisher.Publish("identity.user.registered.v1", msg); err != nil {
	return err
}

return tx.Commit()
```

!!! note
    `store.Append(...)` and `outboxPublisher.Publish(...)` share the same `*sql.Tx`.
    If publish fails, rollback removes both the domain append and staged outbox message.

## Forwarding to RabbitMQ (example)

Run forwarding as a dedicated worker process so broker outages or retries do not block request paths.

### Forwarder worker wiring snippet (Go)

```go
package outboxworker

import (
	"context"
	stdsql "database/sql"

	"github.com/ThreeDotsLabs/watermill"
	"github.com/ThreeDotsLabs/watermill-amqp/v3/pkg/amqp"
	"github.com/ThreeDotsLabs/watermill/components/forwarder"
	wmsql "github.com/ThreeDotsLabs/watermill-sql/v4/pkg/sql"
)

func Run(ctx context.Context, db *stdsql.DB, rabbitURI string) error {
	logger := watermill.NewStdLogger(false, false)

	sqlSubscriber, err := wmsql.NewSubscriber(
		wmsql.BeginnerFromStdSQL(db),
		wmsql.SubscriberConfig{
			SchemaAdapter:    wmsql.DefaultPostgreSQLSchema{},
			OffsetsAdapter:   wmsql.DefaultPostgreSQLOffsetsAdapter{},
			InitializeSchema: true,
		},
		logger,
	)
	if err != nil {
		return err
	}
	defer sqlSubscriber.Close()

	rabbitPublisher, err := amqp.NewPublisher(
		amqp.NewDurableQueueConfig(rabbitURI),
		logger,
	)
	if err != nil {
		return err
	}
	defer rabbitPublisher.Close()

	fwd, err := forwarder.NewForwarder(
		sqlSubscriber,
		rabbitPublisher,
		logger,
		forwarder.Config{
			ForwarderTopic: "outbox.forwarder",
		},
	)
	if err != nil {
		return err
	}

	return fwd.Run(ctx)
}
```

## Delivery guarantees and ordering

Outbox + Forwarder gives **at-least-once** delivery.

!!! warning
    Duplicate delivery is expected. All consumers of integration messages must be idempotent.

Practical implications:

- The same message may be delivered more than once.
- Retries happen during transient SQL, network, or broker failures.
- Ordering is never globally guaranteed across all message types.
- Keep ordering assumptions narrow (for example, per aggregate ID or per routing key).

## Poison queue: why it is necessary

Forwarding to RabbitMQ does not eliminate downstream failures. Handlers can still fail repeatedly due to bad payloads, schema mismatches, or business rule violations.

A poison queue captures messages that exhausted retries so operations can inspect and re-drive safely.

Typical flow:

1. Retry middleware handles transient errors.
2. Persistent failures are routed to poison queue.
3. Operators inspect/fix/re-drive.

Minimal middleware wiring example:

```go
poisonMw, err := middleware.PoisonQueue(rabbitPublisher, "orders.poison")
if err != nil {
	return err
}

router.AddMiddleware(
	middleware.Retry{
		MaxRetries:      5,
		InitialInterval: 200 * time.Millisecond,
		MaxInterval:     5 * time.Second,
		Logger:          logger,
	}.Middleware,
	poisonMw,
)
```

References:

- Forwarder docs: <https://watermill.io/advanced/forwarder/>
- SQL Pub/Sub docs: <https://watermill.io/pubsubs/sql/>
- Middleware docs (retry/poison): <https://watermill.io/docs/middlewares/>

## Operational checklist

- Monitor staged backlog size.
- Monitor age of the oldest staged message.
- Alert on forwarder errors and sustained retry spikes.
- Define retention/purge policy for forwarded SQL rows.
- Keep a documented poison re-drive runbook.

### SQL backlog query example

```sql
-- Replace outbox_messages with your configured SQL messages table.
SELECT COUNT(*) AS staged_messages FROM outbox_messages;

SELECT NOW() - MIN(created_at) AS oldest_message_age
FROM outbox_messages;
```

### Purge example

```sql
-- Keep only recent history once messages are fully forwarded and operationally safe to remove.
DELETE FROM outbox_messages
WHERE created_at < NOW() - INTERVAL '7 days';
```

## Troubleshooting

### Forwarder not moving messages

- Verify forwarder process is running.
- Verify `ForwarderTopic` matches publisher and forwarder config.
- Check SQL subscriber offsets table for stalled progress.
- Check broker connection/authentication and network egress.

### SQL table growing too fast

- Confirm forwarder can keep up (CPU/network/broker limits).
- Confirm retry storms are not reprocessing the same failures endlessly.
- Add/adjust retention purge jobs after messages are safely forwarded.

### Duplicate message symptoms

- Confirm duplicates are handled by idempotency keys in consumers.
- Ensure retry + poison behavior is intentional and observable.
- Avoid side effects before idempotency checks.

### Broker outage or backpressure

- Expect SQL backlog to grow during outage windows.
- Scale forwarder workers after broker recovery to drain backlog.
- Keep alerts on backlog age, not only backlog size.
