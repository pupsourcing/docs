# Getting Started

This guide covers installation, setup, and creating your first event-sourced application with pupsourcing.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Complete Example](#complete-example)
- [Next Steps](#next-steps)
- [Common Patterns](#common-patterns)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- Go 1.23 or later
- PostgreSQL 12+ (or SQLite for development/testing)

## Installation

```bash
go get github.com/getpup/pupsourcing
```

Choose a database driver:
```bash
# PostgreSQL (recommended for production)
go get github.com/lib/pq

# SQLite (suitable for development/testing)
go get modernc.org/sqlite

# MySQL/MariaDB
go get github.com/go-sql-driver/mysql
```

## Quick Start

### 1. Generate Database Schema

Generate SQL migrations for your chosen database:

```bash
go run github.com/getpup/pupsourcing/cmd/migrate-gen -output migrations
```

Or use `go generate`:

```go
//go:generate go run github.com/getpup/pupsourcing/cmd/migrate-gen -output migrations
```

This creates SQL migration files with:
- Events table with proper indexes
- Aggregate heads table for version tracking
- Projection checkpoints table

### 2. Apply Schema Migrations

Apply the generated migrations using your preferred migration tool (golang-migrate, goose, etc.).

### 3. Initialize Database Connection

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

db, err := sql.Open("postgres", 
    "host=localhost port=5432 user=postgres password=postgres dbname=myapp sslmode=disable")
if err != nil {
    log.Fatal(err)
}
defer db.Close()
```

### 4. Create Event Store

```go
import (
    "github.com/getpup/pupsourcing/es/adapters/postgres"
)

store := postgres.NewStore(postgres.DefaultStoreConfig())
```

### 5. Append Your First Event

```go
import (
    "github.com/getpup/pupsourcing/es"
    "github.com/google/uuid"
    "time"
)

// Define your event payload
type UserCreated struct {
    Email string `json:"email"`
    Name  string `json:"name"`
}

// Marshal to JSON
payload, _ := json.Marshal(UserCreated{
    Email: "alice@example.com",
    Name:  "Alice Smith",
})

// Create event
aggregateID := uuid.New().String() // In practice, this comes from your domain/business logic
events := []es.Event{
    {
        BoundedContext: "Identity",  // Required: scope events to bounded context
        AggregateType:  "User",
        AggregateID:    aggregateID,
        EventID:        uuid.New(),
        EventType:      "UserCreated",
        EventVersion:   1,
        Payload:        payload,
        Metadata:       []byte(`{}`),
        CreatedAt:      time.Now(),
    },
}

// Append in a transaction
ctx := context.Background()
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

result, err := store.Append(ctx, tx, es.NoStream(), events)
if err != nil {
    log.Fatal(err)
}

if err := tx.Commit(); err != nil {
    log.Fatal(err)
}

fmt.Printf("Event appended at position: %d\n", result.GlobalPositions[0])
fmt.Printf("Aggregate version: %d\n", result.ToVersion())
```

### 6. Read Events

```go
// Read all events for an aggregate
aggregateID := "550e8400-e29b-41d4-a716-446655440000" // Use the actual aggregate ID
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

stream, err := store.ReadAggregateStream(ctx, tx, "Identity", "User", aggregateID, nil, nil)
if err != nil {
    log.Fatal(err)
}

fmt.Printf("Aggregate version: %d\n", stream.Version())

for _, event := range stream.Events {
    fmt.Printf("Event: %s (version %d)\n", event.EventType, event.AggregateVersion)
}
```

### 7. Create a Projection

Create a scoped projection that only receives User events:

```go
import (
    "database/sql"
    "github.com/getpup/pupsourcing/es/projection"
)

type UserCountProjection struct {
    db *sql.DB
}

func (p *UserCountProjection) Name() string {
    return "user_count"
}

// AggregateTypes makes this a scoped projection
func (p *UserCountProjection) AggregateTypes() []string {
    return []string{"User"}  // Only receives User events
}

// BoundedContexts filters by context - receives only Identity context events
func (p *UserCountProjection) BoundedContexts() []string {
    return []string{"Identity"}
}

func (p *UserCountProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    if event.EventType == "UserCreated" {
        // Use the processor's transaction for atomic updates
        // This ensures the read model and checkpoint are updated together
        _, err := tx.ExecContext(ctx,
            "INSERT INTO user_stats (metric, value) VALUES ('total_users', 1) "+
            "ON CONFLICT (metric) DO UPDATE SET value = user_stats.value + 1")
        if err != nil {
            return err
        }
        
        // Query current count for logging
        var count int
        err = tx.QueryRowContext(ctx, 
            "SELECT value FROM user_stats WHERE metric = 'total_users'").Scan(&count)
        if err == nil {
            fmt.Printf("User count: %d\n", count)
        }
    }
    return nil
}
```

### 8. Run the Projection

```go
proj := &UserCountProjection{}
config := projection.DefaultProcessorConfig()

// Use adapter-specific processor
store := postgres.NewStore(postgres.DefaultStoreConfig())
processor := postgres.NewProcessor(db, store, &config)

ctx, cancel := context.WithCancel(context.Background())
defer cancel()

// Run until context is cancelled
err := processor.Run(ctx, proj)
```

!!! tip "Production recommendation (v1.4.0+) for multiple projections"
    If you run multiple continuous projections in the same process, start a shared `projection.Dispatcher` and set `config.WakeupSource = dispatcher` for each processor.  
    This reduces redundant idle polling and database load, while fallback polling still guarantees catch-up correctness.
    
    See the runnable example: [`examples/dispatcher-runner`](https://github.com/getpup/pupsourcing/tree/master/examples/dispatcher-runner).

!!! tip "Testing Projections"
    When writing integration tests for projections, use `RunModeOneOff` to process events synchronously:
    
    ```go
    config := projection.DefaultProcessorConfig()
    config.RunMode = projection.RunModeOneOff
    ```
    
    This allows your tests to process all events and exit cleanly, making assertions straightforward. See the [One-Off Projection Processing](./projections.md#one-off-projection-processing) guide for complete examples.

## Complete Example

See the [complete working example](https://github.com/getpup/pupsourcing/tree/master/examples/single-worker/main.go) that ties everything together.

## Next Steps

- Learn [Core Concepts](./core-concepts.md) to understand event sourcing with pupsourcing
- Explore [Projections & Scaling](./scaling.md) to build read models
- See [Scaling Guide](./scaling.md) for production deployments
- Browse [Examples](https://github.com/getpup/pupsourcing/tree/master/examples) for more patterns

## Common Patterns

### Appending Multiple Events

```go
userID := uuid.New().String()
events := []es.Event{
    {
        BoundedContext: "Identity",
        AggregateType:  "User",
        AggregateID:    userID,
        EventID:        uuid.New(),
        EventType:      "UserCreated",
        EventVersion:   1,
        Payload:        payload1,
        Metadata:       []byte(`{}`),
        CreatedAt:      time.Now(),
    },
    {
        BoundedContext: "Identity",
        AggregateType:  "User",
        AggregateID:    userID,  // Same aggregate
        EventID:        uuid.New(),
        EventType:      "EmailVerified",
        EventVersion:   1,
        Payload:        payload2,
        Metadata:       []byte(`{}`),
        CreatedAt:      time.Now(),
    },
}

// Both events appended atomically
result, err := store.Append(ctx, tx, es.NoStream(), events)
```

### Handling Version Conflicts

```go
result, err := store.Append(ctx, tx, es.Exact(currentVersion), events)
if errors.Is(err, store.ErrOptimisticConcurrency) {
    // Another transaction modified this aggregate
    // Retry the entire operation
    tx.Rollback()
    // ... retry logic
}
```

### Reading Event Ranges

```go
aggregateID := uuid.New().String()

// Read from version 5 onwards (e.g., after loading a snapshot)
fromVersion := int64(5)
stream, err := store.ReadAggregateStream(ctx, tx, "Identity", "User", aggregateID, &fromVersion, nil)

// Read a specific range
toVersion := int64(10)
stream, err := store.ReadAggregateStream(ctx, tx, "Identity", "User", aggregateID, &fromVersion, &toVersion)
```

## Troubleshooting

### Connection Errors

Ensure PostgreSQL is running:
```bash
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:16
```

### Migration Issues

Verify migrations were applied:
```shell
\d events
\d aggregate_heads
\d projection_checkpoints
```

### Event Not Appearing

Check transaction was committed:
```go
tx, _ := db.BeginTx(ctx, nil)
store.Append(ctx, tx, es.NoStream(), events)
tx.Commit() // Don't forget this!
```

### Projection Not Processing

Verify events exist:
```sql
SELECT COUNT(*) FROM events;
```

Check projection checkpoint:
```sql
SELECT * FROM projection_checkpoints WHERE projection_name = 'your_projection';
```

## Resources

- [Core Concepts](./core-concepts.md) - Understand the fundamentals
- [API Reference](./api-reference.md) - Complete API documentation
- [Examples](https://github.com/getpup/pupsourcing/tree/master/examples) - Working code examples
