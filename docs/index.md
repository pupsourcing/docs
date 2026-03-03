# Pupsourcing

Event Sourcing infrastructure for Go that handles the complexity without creeping into your domain model.

---

## Quick Start

### 1. Install

```bash
go get github.com/pupsourcing/core
```

Choose your database driver:

```bash
# PostgreSQL
go get github.com/lib/pq
```

Generate database schema:

```bash
# Generate SQL migrations for your database
go run github.com/pupsourcing/core/cmd/migrate-gen -output migrations

# Load the generated SQL into your database
```

### 2. Save an Event

```go
import (
    "github.com/pupsourcing/core/es"
    "github.com/pupsourcing/core/es/adapters/postgres"
)

// Create event store
store := postgres.NewStore(postgres.DefaultStoreConfig())

// Define and save your event
event := es.Event{
    BoundedContext: "Identity",
    AggregateType:  "User",
    AggregateID:    userID,
    EventType:      "UserCreated",
    EventVersion:   1,
    Payload:        payload,
}

tx, _ := db.BeginTx(ctx, nil)
result, _ := store.Append(ctx, tx, es.NoStream(), []es.Event{event})
tx.Commit()
```

### 3. Read a Stream

```go
// Read all events for an aggregate
stream, err := store.ReadAggregateStream(
    ctx, tx,
    "Identity",  // bounded context
    "User",      // aggregate type
    userID,      // aggregate ID
    nil, nil,    // version range
)

// Process events
for _, event := range stream.Events {
    // Apply event to rebuild state
}
```

---

## Why Pupsourcing?

**Mature Infrastructure** — Battle-tested event sourcing that handles persistence, concurrency, and consumers without touching your domain logic.

**Simple to Start** — Get running quickly with sensible defaults, then unlock powerful capabilities like auto-scaling workers and temporal queries.

**Go Native** — Pure Go with idiomatic APIs. No annotations, no magic—just clean, explicit code.

---

## Learn More

- **[Getting Started](getting-started.md)** — Complete setup guide
- **[Documentation](docs-overview.md)** — Deep dive into concepts and patterns
- **[GitHub](https://github.com/pupsourcing/core)** — Source code and examples
- **[Integration Testing](https://github.com/pupsourcing/core/tree/master/examples/integration-testing)** — Learn how to test consumers
