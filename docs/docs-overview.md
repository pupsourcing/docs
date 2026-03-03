# Pupsourcing

A production-ready Event Sourcing library for Go with clean architecture principles.

---

## What is Event Sourcing?

Event sourcing is a powerful architectural pattern that stores **state changes as an immutable sequence of events** rather than maintaining only the current state. Instead of updating records (CRUD), your system appends events that describe **what happened**.

### Think of it Like a Bank Statement

Imagine your bank account. The bank doesn't just store your current balance - they keep a complete record of every transaction:

```
Jan 1:  Deposit    +$1000  → Balance: $1000
Jan 5:  Withdraw   -$200   → Balance: $800
Jan 10: Deposit    +$500   → Balance: $1300
Jan 15: Withdraw   -$300   → Balance: $1000
```

If you wanted to know your balance on January 10th, the bank could replay all transactions up to that date. This is exactly how event sourcing works - instead of storing the final balance, you store every transaction (event) and calculate the current state by replaying them.

### How Does This Work in Software?

In event sourcing, you never update or delete data. Instead, you:

1. **Write events** when something happens (UserRegistered, EmailChanged, OrderPlaced)
2. **Store events** in an append-only log (events can't be changed or deleted)
3. **Read events** and replay them to reconstruct the current state
4. **Build consumers** (including projections/read models) by processing events into query-friendly outputs

### The Traditional Approach vs Event Sourcing

**Traditional CRUD - Updates destroy history:**

```
User table:
┌────┬─────────────────────┬───────┬────────┐
│ id │ email               │ name  │ status │
├────┼─────────────────────┼───────┼────────┤
│ 1  │ alice@example.com   │ Alice │ active │
└────┴─────────────────────┴───────┴────────┘
```

```sql
-- UPDATE loses history - no way to know what changed
UPDATE users SET email='new@email.com' WHERE id=1;
```

**Event Sourcing - Preserves complete history:**

```
Events (append-only log):
┌───────────────────────────────────────────────────────┐
│ 1. UserCreated                                        │
│    {id: 1, email: "alice@example.com", name: "Alice"} │
├───────────────────────────────────────────────────────┤
│ 2. EmailVerified                                      │
│    {id: 1}                                            │
├───────────────────────────────────────────────────────┤
│ 3. EmailChanged                                       │
│    {id: 1, from: "alice@example.com",                 │
│     to: "alice@newdomain.com"}                        │
├───────────────────────────────────────────────────────┤
│ 4. UserDeactivated                                    │
│    {id: 1, reason: "account closed"}                  │
└───────────────────────────────────────────────────────┘

Current state = Apply events 1-4 in sequence
Historical state = Apply events up to any point in time
```

### Real-World Example: E-commerce Order

Let's see a concrete example of how event sourcing works with an online shopping order:

```go
// Traditional approach - single row gets updated repeatedly
Order {
    ID: "order-123",
    Status: "delivered",  // Lost history: was it created? paid? shipped?
    Items: [...],
    Total: 99.99
}

// Event sourcing - complete audit trail
Events for order-123:
1. OrderCreated      { items: [...], total: 99.99 }
2. PaymentProcessed  { method: "credit_card", amount: 99.99 }
3. OrderShipped      { carrier: "FedEx", tracking: "123456789" }
4. OrderDelivered    { deliveredAt: "2024-01-15T14:30:00Z" }

// Replay events to get current state
Order = empty order object

FOR EACH event IN events:
    IF event is OrderCreated:
        Order.Items = event.items
        Order.Total = event.total
        Order.Status = "created"
    ELSE IF event is PaymentProcessed:
        Order.Status = "paid"
    ELSE IF event is OrderShipped:
        Order.Status = "shipped"
        Order.TrackingNumber = event.tracking
    ELSE IF event is OrderDelivered:
        Order.Status = "delivered"

// Result: Final state
Order {
    Items: [...],
    Total: 99.99,
    Status: "delivered",
    TrackingNumber: "123456789"
}
```

### How to Query Orders?

This is a common question for newcomers: "If everything is stored as events, how do I get a simple list of orders?"

The answer is **consumers**. A projection is a consumer that builds read models optimized for queries:

**Events (append-only):**
```
1. OrderCreated      { id: 1, items: [...], total: 99.99 }
2. OrderCreated      { id: 2, items: [...], total: 49.99 }
3. PaymentProcessed  { id: 1, method: "credit_card" }
4. OrderShipped      { id: 1, tracking: "123456" }
5. OrderDelivered    { id: 2 }
```

**Read model built by a projection consumer (`orders_view` table):**
```
Process each event and update a regular database table:
┌────┬────────┬──────────────┬────────┐
│ id │ total  │ tracking     │ status │
├────┼────────┼──────────────┼────────┤
│ 1  │ 99.99  │ 123456       │ shipped│
│ 2  │ 49.99  │ -            │ deliver│
└────┴────────┴──────────────┴────────┘
```

Now you can query: `SELECT * FROM orders_view WHERE status = 'shipped'` - fast and simple!

**Key insight:** You keep both the events (for history and replaying) and consumer outputs (for fast queries and integrations). Projection read models are built from events and can be rebuilt at any time.

### Why Event Sourcing?

Event sourcing provides powerful capabilities that are difficult or impossible with traditional CRUD:

**✅ Complete Audit Trail**
- Every state change is recorded with full context
- Perfect for compliance (financial, healthcare, legal)
- Natural debugging: see exactly what happened and when

**✅ Temporal Queries**
- "What was the user's email on January 1st?"
- "Show me all orders that were pending last week"
- Reconstruct past state at any point in time

**✅ Flexible Read Models**
- Build new views from existing events without migrations
- Multiple consumers from the same event stream
- Add new read models without touching write side

**✅ Event Replay**
- Fix bugs by replaying events with corrected logic
- Test new features on production data
- Generate new read models from historical events

**✅ Business Intelligence**
- Rich analytics from complete event history
- Answer questions that weren't anticipated
- "How many users changed their email in the last month?"

### When to Use Event Sourcing

**✅ Great fit:**

- Systems requiring audit trails (finance, healthcare, legal)
- Complex business domains with rich behavior
- Applications needing temporal queries
- Microservices publishing domain events
- Multiple read models from the same data

**ℹ️ Consideration:**

- Event sourcing usually introduces eventual consistency between writes and read models, so confirm brief staleness is acceptable for key workflows.
- It comes with a learning curve (event modeling, replay, projections, and operations), especially for teams adopting it for the first time.
- Familiarity with bounded contexts and Domain-Driven Design (DDD) generally improves outcomes by making boundaries and event semantics clearer.

---

## How Pupsourcing Helps

Pupsourcing makes event sourcing in Go **simple, clean, and production-ready**. Here's what sets it apart:

### 🎯 Clean Architecture

No infrastructure creep into your domain model:

```go
// Your domain events are plain Go structs
type UserCreated struct {
    Email string
    Name  string
}

// No annotations, no framework inheritance
// Pure domain logic
```

### 🔌 PostgreSQL Native

Built for PostgreSQL with efficient storage and querying:

```go
// PostgreSQL
store := postgres.NewStore(postgres.DefaultStoreConfig())
```

### 🏗️ Bounded Context Support

Align with Domain-Driven Design (DDD):

```go
// Events are scoped to bounded contexts
event := es.Event{
    BoundedContext: "Identity",  // Clear domain boundaries
    AggregateType:  "User",
    AggregateID:    userID,
    EventType:      "UserCreated",
    // ...
}
```

### 🔒 Optimistic Concurrency

Automatic conflict detection prevents lost updates:

```go
// Append with expected version
result, err := store.Append(ctx, tx, 
    es.Exact(3),  // Expects version 3
    []es.Event{event},
)
// If another process already wrote version 4, this fails
```

### 📊 Powerful Consumers

Transform events into query-optimized read models:

```go
// Scoped consumer (projection) - only User events from Identity context
type UserReadModel struct{}

func (p *UserReadModel) AggregateTypes() []string {
    return []string{"User"}
}

func (p *UserReadModel) BoundedContexts() []string {
    return []string{"Identity"}
}

func (p *UserReadModel) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Update your read model using the provided transaction
    switch event.EventType {
    case "UserCreated":
        // Use tx for atomic updates
        _, err := tx.ExecContext(ctx, "INSERT INTO user_read_model ...")
        return err
    case "EmailChanged":
        // All changes in the same transaction as checkpoint
        _, err := tx.ExecContext(ctx, "UPDATE user_read_model ...")
        return err
    }
    return nil
}
```

### 📈 Horizontal Scaling

Built-in support for auto-scaling consumers across multiple workers:

```go
w := postgres.NewWorker(db, store, worker.WithTotalSegments(4))
err := w.Run(ctx, &UserReadModel{}, &OrderProjection{})
```

### 🛠️ Code Generation

Optional type-safe event mapping:

```bash
# Generate strongly-typed event mappers
go run github.com/pupsourcing/core/cmd/eventmap-gen \
  -input internal/domain/events \
  -output internal/infrastructure/generated
```

### 🎁 Minimal Dependencies

- Go standard library
- Database driver (your choice)
- That's it!

---

## Quick Start

### Installation

```bash
go get github.com/pupsourcing/core

# PostgreSQL driver
go get github.com/lib/pq
```

### Your First Event

```go
import (
    "github.com/pupsourcing/core/es"
    "github.com/pupsourcing/core/es/adapters/postgres"
    "github.com/google/uuid"
)

// Create store
store := postgres.NewStore(postgres.DefaultStoreConfig())

// Create event
event := es.Event{
    BoundedContext: "Identity",
    AggregateType:  "User",
    AggregateID:    uuid.New().String(),
    EventID:        uuid.New(),
    EventType:      "UserCreated",
    EventVersion:   1,
    Payload:        []byte(`{"email":"alice@example.com","name":"Alice"}`),
    Metadata:       []byte(`{}`),
    CreatedAt:      time.Now(),
}

// Append to event store
tx, _ := db.BeginTx(ctx, nil)
result, err := store.Append(ctx, tx, es.NoStream(), []es.Event{event})
if err != nil {
    tx.Rollback()
    log.Fatal(err)
}
tx.Commit()

fmt.Printf("Event stored at position: %d\n", result.GlobalPositions[0])
```

### Read Events

```go
// Read all events for an aggregate
stream, err := store.ReadAggregateStream(
    ctx, tx, 
    "Identity",  // bounded context
    "User",      // aggregate type
    aggregateID, // aggregate ID
    nil, nil,    // from/to version
)

// Process events
for _, event := range stream.Events {
    fmt.Printf("Event: %s at version %d\n", 
        event.EventType, event.AggregateVersion)
}
```

---

## Documentation Structure

### Getting Started
Start here if you're new to pupsourcing:

- **[Getting Started](getting-started.md)** - Installation, setup, and your first event-sourced application
  - Prerequisites and installation
  - Database schema generation
  - Creating and storing your first event
  - Reading events back
  - Building a simple consumer

### Core Concepts
Understand the fundamentals:

- **[Core Concepts](core-concepts.md)** - Deep dive into event sourcing principles with pupsourcing
  - Event sourcing fundamentals
  - Core components (Events, Aggregates, Event Store, Consumers)
  - Key concepts (Optimistic Concurrency, Global Position, Idempotency)
  - Design principles (Library vs Framework, Explicit Dependencies)
  - Common patterns (Read-Your-Writes, Event Upcasting, Aggregate Reconstruction)

### Database Adapter
Configure your PostgreSQL database:

- **[Database Adapter](adapters.md)** - PostgreSQL adapter documentation
  - PostgreSQL (production-ready)
  - Configuration options and performance tuning

### Consumers
Build read models and integration handlers:

- **[Consumers](consumers.md)** - Building and running consumers
  - Consumer interface and scoped consumers
  - Projections as a specific consumer type
  - Worker API for continuous workloads
  - One-off processing with BasicProcessor

### Code Generation
Type-safe event mapping:

- **[Event Mapping Code Generation](eventmap-gen.md)** - Strongly-typed conversion between domain and ES events
  - Why this tool exists
  - Installation and quick start
  - Versioned events (schema evolution)
  - Clean architecture integration
  - Repository adapter pattern

### Production Operations
Deploy and monitor:

- **[Deployment](deployment.md)** - Production deployment patterns and operational best practices
  - Worker-first deployment approach
  - Auto-scaling behavior and segment ownership
  - Deployment examples
  - Monitoring and metrics
  - Graceful shutdown
  - Troubleshooting

- **[Observability](observability.md)** - Logging, tracing, and monitoring
  - Logger interface and integration
  - Distributed tracing (TraceID, CorrelationID, CausationID)
  - Metrics integration (Prometheus examples)
  - Best practices

### About
Learn more about the project:

- **[About](about.md)** - Project philosophy, history, and community

---

## Quick Links

### For Beginners
1. Start with [Getting Started](getting-started.md)
2. Understand [Core Concepts](core-concepts.md)
3. Explore [Examples](https://github.com/pupsourcing/core/tree/master/examples)

### For Production
1. Review [Database Adapter](adapters.md) and configure PostgreSQL
2. Implement [Consumers](consumers.md)
3. Set up [Observability](observability.md)
4. Follow [Deployment Guide](deployment.md)

### For Advanced Users
1. Implement [Event Mapping Code Generation](eventmap-gen.md)
2. Tune worker configuration in [Deployment](deployment.md)
3. Build specialized consumers in [Consumers](consumers.md)

---

## Examples Repository

The [pupsourcing repository](https://github.com/pupsourcing/core) includes comprehensive working examples:

- **basic** - Complete PostgreSQL example
- **worker** - Worker API with segment-based auto-scaling
- **integration-testing** - One-off processing pattern for tests
- **stop-resume** - Checkpoint management and resumption
- **with-logging** - Observability integration
- **eventmap-codegen** - Type-safe event mapping generation

Each example includes a README with setup instructions and explanation of concepts.

---

## Community and Support

- **GitHub Repository**: [github.com/pupsourcing/core](https://github.com/pupsourcing/core)
- **Issues & Discussions**: [GitHub Issues](https://github.com/pupsourcing/core/issues)
- **Documentation**: [pupsourcing.gopup.dev](https://pupsourcing.gopup.dev)

---

## What's Next?

- **[Getting Started](getting-started.md)** - Complete setup guide and first steps
- **[Core Concepts](core-concepts.md)** - Deep dive into event sourcing principles
- **[Database Adapter](adapters.md)** - Configuring your PostgreSQL database
- **[Consumers](consumers.md)** - Building read models and integration consumers
- **[Deployment](deployment.md)** - Worker-first production guidance

---

## Production Ready

Pupsourcing is designed for production use with:

- **Comprehensive test coverage** - Unit and integration tests
- **Battle-tested patterns** - Based on proven event sourcing principles
- **Clear documentation** - Extensive guides and examples
- **Active maintenance** - Regular updates and bug fixes
- **Clean codebase** - Easy to understand and extend

---

## License

This project is licensed under the MIT License - see the [LICENSE](https://github.com/pupsourcing/core/blob/master/LICENSE) file for details.
