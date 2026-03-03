# Consumers

Guide to building and running consumers in pupsourcing.

## Table of Contents

1. [Consumers Overview](#consumers-overview)
2. [Consumer Interface](#consumer-interface)
3. [Projections as Scoped Consumers](#projections-as-scoped-consumers)
4. [Running Consumers with Worker](#running-consumers-with-worker)
5. [Worker Configuration](#worker-configuration)
6. [Wakeup Sources](#wakeup-sources)
7. [One-Off Consumer Processing](#one-off-consumer-processing)
8. [Integration Testing with RunModeOneOff](#integration-testing-with-runmodeoneoff)
9. [Consumer Guarantees](#consumer-guarantees)

## Consumers Overview

A **consumer** processes persisted events and advances a checkpoint.

A **projection** is a specific type of consumer that builds query-optimized read models.

```text
Events -> Consumer Handle() -> Side effects (SQL read model, broker publish, cache, etc.)
```

Use consumers for:

- Read models and query tables
- Outbox/integration publishing
- Audit pipelines
- Derived metrics and materialized views

## Consumer Interface

Consumers implement `consumer.Consumer`:

```go
package myapp

import (
    "context"
    "database/sql"

    "github.com/pupsourcing/core/es"
)

type UserProjection struct{}

func (p *UserProjection) Name() string {
    return "user_projection"
}

func (p *UserProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Use tx for SQL read models stored in the same database.
    // The processor commits/rolls back tx.
    _, err := tx.ExecContext(ctx,
        "INSERT INTO user_events (event_id, event_type) VALUES ($1, $2) ON CONFLICT (event_id) DO NOTHING",
        event.EventID,
        event.EventType,
    )
    return err
}
```

## Projections as Scoped Consumers

For read models, you typically scope consumers by aggregate type and bounded context.

```go
package myapp

import (
    "context"
    "database/sql"

    "github.com/pupsourcing/core/es"
)

type UserReadModel struct{}

func (p *UserReadModel) Name() string { return "user_read_model" }

func (p *UserReadModel) AggregateTypes() []string {
    return []string{"User"}
}

func (p *UserReadModel) BoundedContexts() []string {
    return []string{"Identity"}
}

func (p *UserReadModel) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    _, err := tx.ExecContext(ctx,
        "INSERT INTO users_view (user_id, event_type, created_at) VALUES ($1, $2, $3)",
        event.AggregateID,
        event.EventType,
        event.CreatedAt,
    )
    return err
}
```

## Running Consumers with Worker

Use the Worker API for continuous consumer processing.

```go
package main

import (
    "context"
    "database/sql"

    "github.com/pupsourcing/core/es/adapters/postgres"
)

func run(ctx context.Context, db *sql.DB) error {
    store := postgres.NewStore(postgres.DefaultStoreConfig())

    w := postgres.NewWorker(db, store)

    return w.Run(ctx,
        &UserReadModel{},
        &OrderProjection{},
        &BillingOutboxConsumer{},
    )
}
```

Each worker instance automatically claims a fair share of segments per consumer.

## Worker Configuration

Customize worker behavior with options:

```go
package main

import (
    "time"

    "github.com/pupsourcing/core/es/adapters/postgres"
    "github.com/pupsourcing/core/es/worker"
)

w := postgres.NewWorker(db, store,
    worker.WithTotalSegments(4),                  // Start conservative; increase only when needed.
    worker.WithBatchSize(200),                   // Processes up to 200 events per fetch.
    worker.WithPollInterval(500*time.Millisecond), // Base polling interval between event checks.
    worker.WithMaxPollInterval(30*time.Second),  // Backs off polling to reduce idle DB load.
    worker.WithHeartbeatInterval(5*time.Second), // Reports liveness while owning segments.
    worker.WithStaleThreshold(30*time.Second),   // Lets others reclaim segments from dead workers.
    worker.WithRebalanceInterval(10*time.Second), // Rebalances segment ownership across workers.
)

err := w.Run(ctx, &UserReadModel{}, &OrderProjection{})
```

`TotalSegments` defines the parallelism ceiling per consumer. Start with 4 and increase only when you have clear evidence of bottlenecks. See [Deployment — Choosing TotalSegments](deployment.md#choosing-totalsegments) for detailed guidance.

## Wakeup Sources

A **wakeup source** tells segment processors when new events are available. Without one, segments rely purely on polling — checking the database at regular intervals. With a wakeup source, segments sleep until a signal arrives, then immediately check for work.

pupsourcing ships with two wakeup source implementations:

### Built-in Polling Dispatcher (default)

The Worker includes a built-in `Dispatcher` that periodically reads the latest global position from the events table and broadcasts a wake signal to all segment processors when it advances.

- **Pros:** Zero configuration, works everywhere
- **Cons:** Adds periodic read queries even when no events are flowing. At idle, the floor is `total_owned_segments / MaxPollInterval` queries per second per worker.

This is enabled by default. You can disable it with `worker.WithEnableDispatcher(false)`.

### NotifyDispatcher (LISTEN/NOTIFY)

The `NotifyDispatcher` uses PostgreSQL's native [LISTEN/NOTIFY](https://www.postgresql.org/docs/current/sql-notify.html) mechanism for instant event notifications. When an event is appended, `pg_notify` fires **inside the same transaction** — so the notification arrives only when the transaction commits, guaranteeing no phantom wakes.

- **Pros:** Zero idle database load, instant wake on event append (sub-millisecond latency)
- **Cons:** Requires a dedicated PostgreSQL connection (managed by `pq.NewListener`, not from your connection pool)

**Setup requires two pieces:**

1. **Store** — configure the NOTIFY channel so `Append()` fires `pg_notify`:

    ```go
    store := postgres.NewStore(postgres.NewStoreConfig(
        postgres.WithNotifyChannel("pupsourcing_events"),
    ))
    ```

2. **Worker** — create a `NotifyDispatcher` and pass it as the wakeup source:

    ```go
    import "github.com/pupsourcing/core/es/worker"

    // connStr is a Postgres connection string (not *sql.DB)
    nd := postgres.NewNotifyDispatcher(connStr, &postgres.NotifyDispatcherConfig{
        Channel:          "pupsourcing_events", // Must match WithNotifyChannel
        FallbackInterval: 30 * time.Second,     // Safety net for missed notifications
    })

    w := postgres.NewWorker(db, store,
        worker.WithWakeupSource(nd),
        worker.WithTotalSegments(4),
    )

    err := w.Run(ctx, &UserReadModel{}, &OrderProjection{})
    ```

**How it works:**

1. `store.Append()` executes `SELECT pg_notify('pupsourcing_events', '<position>')` inside the caller's transaction
2. When the transaction commits, PostgreSQL delivers the notification to all `LISTEN` connections
3. `NotifyDispatcher` receives the notification via `pq.NewListener` and broadcasts a wake signal to all subscribed segment processors
4. Each segment processor immediately reads new events from the store

**Fallback timer:** The `NotifyDispatcher` includes a configurable fallback timer (default 30s). If no notification arrives within this window — due to a connection drop, network partition, or any other transient issue — it sends a wake signal anyway. This guarantees eventual progress even if LISTEN/NOTIFY is temporarily disrupted.

**NotifyDispatcher configuration options:**

| Option | Default | Purpose |
|--------|---------|---------|
| `Channel` | `"pupsourcing_events"` | Postgres NOTIFY channel name |
| `FallbackInterval` | `30s` | Max time without notification before fallback wake |
| `MinReconnectInterval` | `5s` | Min delay before reconnecting after listener error |
| `MaxReconnectInterval` | `60s` | Max delay before reconnecting after repeated errors |
| `Logger` | `nil` | Optional logger for observability |

## One-Off Consumer Processing

Use `BasicProcessor` with `RunModeOneOff` when you want to process available events and exit.

```go
package main

import (
    "github.com/pupsourcing/core/es/consumer"
    "github.com/pupsourcing/core/es/adapters/postgres"
)

store := postgres.NewStore(postgres.DefaultStoreConfig())

cfg := consumer.DefaultBasicProcessorConfig()
cfg.RunMode = consumer.RunModeOneOff

processor := postgres.NewBasicProcessor(db, store, cfg)
err := processor.Run(ctx, &UserReadModel{})
if err != nil {
    return err
}
```

This mode is useful for:

- test runs
- catch-up jobs
- backfills
- controlled replay jobs

## Integration Testing with RunModeOneOff

`RunModeOneOff` makes integration tests deterministic.

```go
func TestUserProjection_OneOff(t *testing.T) {
    db := setupTestDB(t)
    defer db.Close()

    ctx := context.Background()
    store := postgres.NewStore(postgres.DefaultStoreConfig())

    // Arrange: append events
    tx, _ := db.BeginTx(ctx, nil)
    _, err := store.Append(ctx, tx, es.NoStream(), []es.Event{
        {
            BoundedContext: "Identity",
            AggregateType:  "User",
            AggregateID:    "user-1",
            EventID:        uuid.New(),
            EventType:      "UserCreated",
            EventVersion:   1,
            Payload:        []byte(`{"email":"alice@example.com"}`),
            Metadata:       []byte(`{}`),
            CreatedAt:      time.Now(),
        },
    })
    if err != nil {
        t.Fatal(err)
    }
    if err := tx.Commit(); err != nil {
        t.Fatal(err)
    }

    // Act: process and exit
    cfg := consumer.DefaultBasicProcessorConfig()
    cfg.RunMode = consumer.RunModeOneOff

    p := postgres.NewBasicProcessor(db, store, cfg)
    if err := p.Run(ctx, &UserReadModel{}); err != nil {
        t.Fatalf("processor failed: %v", err)
    }

    // Assert read model/checkpoint state
    assertUserExists(t, db, "user-1")
}
```

## Consumer Guarantees

- At-least-once delivery semantics
- Ordered processing per segment
- Atomic checkpoint updates with consumer transaction
- Automatic resume from checkpoints after restart
- Horizontal scaling with Worker on PostgreSQL
