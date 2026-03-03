# Database Adapter

pupsourcing provides a production-ready PostgreSQL adapter that implements all core interfaces for event sourcing operations.

## Table of Contents

- [Architecture](#architecture)
- [PostgreSQL Adapter](#postgresql-adapter)
- [Configuration Options](#configuration-options)
- [Performance Considerations](#performance-considerations)

## Architecture

The adapter implements three core interfaces:

- **`store.EventStore`** - Append events with optimistic concurrency
- **`store.EventReader`** - Sequential event reading by global position
- **`store.AggregateStreamReader`** - Aggregate-specific event retrieval

## PostgreSQL Adapter

**Package:** `github.com/pupsourcing/core/es/adapters/postgres`  
**Driver:** `github.com/lib/pq` (or `github.com/jackc/pgx/v5/stdlib` for pgx benefits)  
**Status:** Production-ready ✅

**Note on drivers:** While the adapter is tested with `github.com/lib/pq`, you can also use the `pgx` driver (`github.com/jackc/pgx/v5/stdlib`) to take advantage of its performance improvements and additional features. Both drivers work with pupsourcing's PostgreSQL adapter.

### Key Features

- Native UUID type for efficient storage and indexing
- JSONB metadata with advanced querying capabilities
- Excellent concurrent write performance
- O(1) version lookups via `aggregate_heads` table
- Optimistic concurrency via unique constraints
- Worker API support (`postgres.NewWorker`) for auto-scaling consumers

### Data Types

| Field | Type | Purpose |
|-------|------|---------|
| `global_position` | `BIGSERIAL` | Auto-incrementing, globally ordered |
| `aggregate_id` | `TEXT` | String-based aggregate identifier (UUIDs, emails, custom IDs) |
| `event_id` | `UUID` | Unique event identifier |
| `payload` | `BYTEA` | Binary event data |
| `metadata` | `JSONB` | Queryable structured metadata |
| `created_at` | `TIMESTAMPTZ` | Timezone-aware timestamp |

### Ideal For

- High-availability production systems
- Multi-tenant applications
- High concurrent write workloads
- Advanced metadata querying requirements

### Usage

```go
import (
    "github.com/pupsourcing/core/es/adapters/postgres"
    _ "github.com/lib/pq"
)

// Basic configuration (recommended for most use cases)
store := postgres.NewStore(postgres.DefaultStoreConfig())

// Advanced configuration with options
config := postgres.NewStoreConfig(
    postgres.WithLogger(myLogger),
    postgres.WithEventsTable("custom_events"),
    postgres.WithCheckpointsTable("custom_checkpoints"),
)
store := postgres.NewStore(config)

// Use with *sql.DB or *sql.Tx
db, _ := sql.Open("postgres", connString)
tx, _ := db.BeginTx(ctx, nil)
result, err := store.Append(ctx, tx, es.NoStream(), events)
tx.Commit()

// Generate migrations
err := migrations.GeneratePostgres(&config)
```

### Consumer Processing

```go
// Recommended for continuous workloads
store := postgres.NewStore(postgres.DefaultStoreConfig())
w := postgres.NewWorker(db, store)
err := w.Run(ctx, myConsumer)
```

```go
// One-off/testing workloads
cfg := consumer.DefaultBasicProcessorConfig()
cfg.RunMode = consumer.RunModeOneOff

processor := postgres.NewBasicProcessor(db, store, cfg)
err := processor.Run(ctx, myConsumer)
```

---

## Configuration Options

```go
type StoreConfig struct {
    // Logger for observability (optional)
    Logger es.Logger
    
    // Table names (customizable)
    EventsTable         string // Default: "events"
    AggregateHeadsTable string // Default: "aggregate_heads"
    SegmentsTable       string // Default: "consumer_segments"
    WorkerRegistryTable string // Default: "consumer_workers"
    
    // NotifyChannel enables LISTEN/NOTIFY for instant event wake-ups (optional)
    NotifyChannel string
}
```

## Performance Considerations

### Write Performance

PostgreSQL provides excellent concurrent write performance. Benefits from connection pooling and proper `max_connections` tuning.

### Read Performance

Leverage JSONB indexes for advanced metadata queries. The `aggregate_heads` table provides O(1) version lookups without scanning the events table.

### Consumer Processing Performance

The Worker API uses segment-based processing for horizontal scaling. See the [Consumers](consumers.md) guide for configuration recommendations.

## Support Matrix

| Go Version | PostgreSQL |
|-----------|-----------|
| 1.23+ | ✅ |
| 1.24+ | ✅ |
| 1.25+ | ✅ |

| Database Version | Support Status |
|-----------------|---------------|
| PostgreSQL 12+ | ✅ Fully supported |
| PostgreSQL 11- | ⚠️ Not tested |

## Examples

Complete working examples are available in the [pupsourcing repository](https://github.com/pupsourcing/core/tree/master/examples):

- **basic** - Complete PostgreSQL example
- **worker** - Worker API with segment-based auto-scaling
- **integration-testing** - One-off processing pattern for tests
- **stop-resume** - Checkpoint management and resumption
- **with-logging** - Observability integration
- **eventmap-codegen** - Type-safe event mapping generation
