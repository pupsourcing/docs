# Deployment & Operations Guide

This guide shows how to run pupsourcing consumers in production using the Worker API.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Running Workers](#running-workers)
- [Scaling Model](#scaling-model)
- [Deployment Examples](#deployment-examples)
- [Worker Configuration](#worker-configuration)
  - [Worker Options](#worker-options)
  - [LISTEN/NOTIFY](#listennotify-recommended)
  - [Database Connection Pool](#database-connection-pool)
- [Operational Checklist](#operational-checklist)
- [Troubleshooting](#troubleshooting)
- [See Also](#see-also)

## Prerequisites

Before deploying workers:

1. Apply generated migrations (including consumer checkpoint + segment tables)
2. Configure database connectivity and connection pool limits
3. Implement your consumers
4. Expose a command or process entrypoint that starts the worker

## Running Workers

Use one process that starts a worker and registers all consumers it should run:

```go
package main

import (
    "context"
    "database/sql"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/pupsourcing/core/es/adapters/postgres"
    "github.com/pupsourcing/core/es/worker"
)

func runConsumers(ctx context.Context, db *sql.DB, connStr string) error {
    store := postgres.NewStore(postgres.NewStoreConfig(
        postgres.WithNotifyChannel("pupsourcing_events"),
    ))

    nd := postgres.NewNotifyDispatcher(connStr, &postgres.NotifyDispatcherConfig{
        Channel:          "pupsourcing_events",
        FallbackInterval: 30 * time.Second,
    })

    w := postgres.NewWorker(
        db,
        store,
        worker.WithWakeupSource(nd),
        worker.WithTotalSegments(4),
        worker.WithBatchSize(200),
        worker.WithPollInterval(500*time.Millisecond),
    )

    return w.Run(ctx,
        &UserProjection{},
        &OrderProjection{},
        &BillingOutboxConsumer{},
    )
}

func main() {
    // initialize db, connStr ...

    ctx, cancel := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer cancel()

    if err := runConsumers(ctx, db, connStr); err != nil {
        panic(err)
    }
}
```

Run the same binary in multiple instances for horizontal scaling.

## Scaling Model

Worker scaling is automatic:

- Each worker instance registers itself in `consumer_workers`
- Workers claim segments from `consumer_segments`
- Rebalancing redistributes segments as workers join/leave
- No partition-key assignment is required

### Choosing TotalSegments

`TotalSegments` defines the upper bound on parallelism **per consumer**. Start conservatively and only increase when you have clear evidence of bottlenecks.

**Recommendation: start with 4 segments.** This is enough for most applications and provides clean scaling up to 4 worker instances. Over-partitioning early causes problems:

- **More coordination overhead.** Each segment has its own checkpoint, heartbeat, and rebalance state. More segments means more database rows to manage and more polling queries at idle.
- **Diminishing returns.** Consumer handlers that touch the same read model tables will contend on row-level locks regardless of how many segments are running. More segments can actually _increase_ lock contention.
- **Harder to reason about.** With 4 segments and 2 workers, each worker gets 2 segments — simple. With 32 segments and 3 workers, you get 11/11/10 splits with constant rebalance chatter.
- **Cannot be changed easily.** There is no automated way to change the number of segments for an existing consumer in a live system yet. Changing `TotalSegments` after deployment requires manual intervention and risks double-processing or missed events. Automated segment split/merge is planned for a future release.

| Total Segments | Worker Instances | Distribution |
|---|---:|---|
| 4 | 1 | 4 |
| 4 | 2 | 2 + 2 |
| 4 | 4 | 1 + 1 + 1 + 1 |
| 4 | 5 | 4 workers active, 1 idle |

If you need more parallelism than 4, increase to 8 (the next power of 2). Avoid arbitrary numbers — powers of 2 distribute more evenly and will be required for future automated segment scaling.

⚠️ **Changing `TotalSegments` for an already-running consumer is not automated yet.** Keep it stable once deployed. Automated segment split/merge (doubling/halving) is on the roadmap.

## Deployment Examples

### Docker Compose

```yaml
version: "3.9"

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp

  worker:
    image: myapp:latest
    depends_on: [postgres]
    environment:
      DATABASE_URL: postgres://postgres:postgres@postgres:5432/myapp?sslmode=disable
    command: ["./myapp", "run-consumers"]
    restart: unless-stopped
    deploy:
      replicas: 2
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-consumers
spec:
  replicas: 4
  selector:
    matchLabels:
      app: myapp-consumers
  template:
    metadata:
      labels:
        app: myapp-consumers
    spec:
      containers:
        - name: worker
          image: myapp:latest
          args: ["run-consumers"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
```

All replicas run the same configuration; workers self-balance segment ownership.

## Worker Configuration

### Worker Options

```go
w := postgres.NewWorker(db, store,
    worker.WithTotalSegments(4),
    worker.WithBatchSize(200),
    worker.WithPollInterval(500*time.Millisecond),
    worker.WithMaxPollInterval(30*time.Second),
    worker.WithWakeupJitter(100*time.Millisecond),
    worker.WithHeartbeatInterval(5*time.Second),
    worker.WithStaleThreshold(30*time.Second),
    worker.WithRebalanceInterval(10*time.Second),
)
```

Option meanings and tradeoffs:

- **TotalSegments**: Number of logical segments per consumer. Start with **4** — see
  [Choosing TotalSegments](#choosing-totalsegments) above.
- **BatchSize**: Maximum events processed per polling cycle per claimed segment. Larger batches
  improve throughput and reduce round trips; smaller batches shorten transaction time and improve
  fairness across segments.
- **PollInterval**: Base wait when a poll finds no work. `500ms` is a good starting point — low
  enough to keep latency acceptable, high enough to avoid hammering the database.
- **MaxPollInterval**: Upper bound for idle backoff between polls. `30s` keeps idle load near zero.
  If you use LISTEN/NOTIFY (see [Wakeup Sources](consumers.md#wakeup-sources)), idle polling almost
  never fires, making this less critical.
- **WakeupJitter**: Random jitter added to segment wakeups to spread load. `100ms` prevents
  all segments from hitting the database simultaneously when a notification arrives.
- **HeartbeatInterval**: Frequency of worker heartbeat updates in `consumer_workers`. `5s` is the
  default and works well for most deployments.
- **StaleThreshold**: Heartbeat age that marks a worker as stale and reclaimable. `30s` is safe for
  most environments. Lower values improve failover speed but risk false positives during GC pauses
  or transient network issues.
- **RebalanceInterval**: How often workers attempt ownership convergence. `10s` balances
  responsiveness with low churn.

### LISTEN/NOTIFY (recommended)

For production deployments, use `NotifyDispatcher` to eliminate idle polling entirely. See
[Wakeup Sources](consumers.md#wakeup-sources) for setup instructions.

```go
store := postgres.NewStore(postgres.NewStoreConfig(
    postgres.WithNotifyChannel("pupsourcing_events"),
))

nd := postgres.NewNotifyDispatcher(connStr, &postgres.NotifyDispatcherConfig{
    Channel:              "pupsourcing_events",
    FallbackInterval:     30 * time.Second,
    MinReconnectInterval: 5 * time.Second,
    MaxReconnectInterval: 60 * time.Second,
})

w := postgres.NewWorker(db, store,
    worker.WithWakeupSource(nd),
    worker.WithTotalSegments(4),
    worker.WithPollInterval(500*time.Millisecond),
    worker.WithMaxPollInterval(30*time.Second),
    worker.WithWakeupJitter(100*time.Millisecond),
)
```

### Database Connection Pool

Properly sized connection pools prevent exhaustion and reduce idle connections:

```go
db.SetMaxOpenConns(15)           // Cap total connections per worker process
db.SetMaxIdleConns(5)            // Keep a small warm pool
db.SetConnMaxIdleTime(5 * time.Minute)  // Reclaim idle connections
db.SetConnMaxLifetime(30 * time.Minute) // Rotate connections periodically
```

**Why these values:**

- **MaxOpenConns(15)**: Each worker runs multiple segment processors, heartbeat loops, and
  rebalance operations concurrently. 15 connections provides enough headroom without exhausting
  the server when running multiple worker replicas. With 4 workers × 15 = 60 connections, which
  fits comfortably in PostgreSQL's default `max_connections=100`.
- **MaxIdleConns(5)**: Keeps a warm pool of ready connections for burst traffic, but doesn't hoard
  connections during quiet periods.
- **ConnMaxIdleTime(5min)**: Reclaims connections that have been idle for 5 minutes. This prevents
  stale connections from accumulating while still avoiding constant reconnection churn.
- **ConnMaxLifetime(30min)**: Forces periodic connection rotation. This helps with DNS-based
  failover, load balancer draining, and prevents issues with long-lived connections to PostgreSQL.

⚠️ The `NotifyDispatcher` uses its own dedicated connection (via `pq.NewListener`) that is **not**
drawn from this pool. Factor in 1 extra connection per worker process when sizing `max_connections`.

## Operational Checklist

- Monitor consumer lag from checkpoints (`last_global_position` vs max event position)
- Set safe DB pool limits for your workload (see [Database Connection Pool](#database-connection-pool) above)
- Ensure consumers use the provided transaction when writing SQL read models
- Run graceful shutdown via context cancellation/SIGTERM

### PostgreSQL Required

The Worker API requires PostgreSQL. Ensure you have PostgreSQL 12+ configured and the generated migrations applied.

## Troubleshooting

### Connection exhaustion

Symptoms include workers timing out on DB calls, `too many connections` errors, or frequent
reconnects.

Check both sides of the limit:

- **Client-side pool settings** (`database/sql`): verify `SetMaxOpenConns`, `SetMaxIdleConns`,
  `SetConnMaxLifetime`, and `SetConnMaxIdleTime` are set for your expected worker/process count.
  Oversized pools across replicas can exhaust the server quickly.
- **Database server limits**: confirm PostgreSQL `max_connections` is high enough for all apps,
  workers, and admin sessions, including reserved slots.

### Workers not claiming segments

Check active workers:

```sql
SELECT consumer_name, worker_id, last_heartbeat
FROM consumer_workers
ORDER BY consumer_name, worker_id;
```

Check segment ownership:

```sql
SELECT consumer_name, segment_id, owner_id, checkpoint
FROM consumer_segments
ORDER BY consumer_name, segment_id;
```

### Consumer lag increasing

Compare checkpoint with latest global position:

```sql
SELECT c.consumer_name,
       c.last_global_position,
       (SELECT MAX(global_position) FROM events) AS latest_position,
       (SELECT MAX(global_position) FROM events) - c.last_global_position AS lag
FROM consumer_checkpoints c
ORDER BY lag DESC;
```

Increase throughput by tuning batch size, database resources, and consumer handler latency.

### Uneven distribution after scaling event

Segment ownership converges on rebalance intervals. Confirm:

- heartbeat updates are current
- stale threshold is reasonable
- enough time has passed for rebalance cycle(s)

## See Also

- [Getting Started](./getting-started.md)
- [Core Concepts](./core-concepts.md)
- [Consumers](./consumers.md)
- [Database Adapters](./adapters.md)
- [Observability](./observability.md)
