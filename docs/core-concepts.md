# Core Concepts

Fundamental principles of event sourcing with pupsourcing.

## Table of Contents

- [Event Sourcing Fundamentals](#event-sourcing-fundamentals)
- [Core Components](#core-components)
- [Key Concepts](#key-concepts)
- [Design Principles](#design-principles)
- [Common Patterns](#common-patterns)
- [See Also](#see-also)

## Event Sourcing Fundamentals

### Definition

Event sourcing stores state changes as an immutable sequence of events rather than maintaining only current state. Instead of updating records (CRUD), the system appends events that describe what happened.

**Traditional CRUD:**
```
User table:
| id | email              | name  | status |
| 1  | alice@example.com  | Alice | active |
```

```sql
# UPDATE loses history
UPDATE user SET email='new@email.com' WHERE id=1
```

**Event Sourcing:**
```
Events (append-only):
1. UserCreated(id=1, email=alice@example.com, name=Alice)
2. EmailVerified(id=1)
3. EmailChanged(id=1, old=alice@example.com, new=alice@newdomain.com)
4. UserDeactivated(id=1, reason="account closed")

Current state = Apply events 1-4 in sequence
Historical state = Apply events up to specific point in time
```

### Benefits

1. **Complete Audit Trail** - Full history of all changes for compliance and debugging
2. **Temporal Queries** - Reconstruct state at any point in time
3. **Flexible Read Models** - Build new projections from existing events without migrations
4. **Event Replay** - Reprocess historical events for debugging or new features
5. **Business Intelligence** - Rich analytical capabilities from event history

### Trade-offs

**Advantages:**

- Complete historical record of all state changes
- Flexible read models without migrations
- Natural audit logging for compliance
- Temporal query capabilities
- Effective debugging through event replay

**Considerations:**

- Higher complexity than simple CRUD
- Learning curve for team members
- Consumers must handle idempotency
- Eventual consistency in read models
- Storage growth over time
- Schema evolution for immutable events

**When to Use:**

- Systems requiring audit trails (financial, healthcare, legal)
- Complex business domains
- Applications needing temporal queries
- Microservices publishing domain events
- Multiple read models from same data

**When to Avoid:**

- Simple CRUD applications
- Prototypes without event sourcing requirements
- Teams lacking event sourcing experience
- Systems requiring strict low-latency everywhere

## Bounded Contexts

pupsourcing requires all events to belong to a **bounded context**, supporting Domain-Driven Design (DDD) principles. A bounded context is an explicit boundary within which a domain model is defined and applicable.

### Why Bounded Contexts?

- **Domain Isolation**: Different parts of your system (e.g., Identity, Billing, Catalog) can evolve independently
- **Clear Boundaries**: Events are explicitly scoped, preventing accidental mixing of concerns
- **Flexible Consumers**: Scoped consumers can filter by both aggregate type and bounded context
- **Uniqueness**: Event uniqueness is enforced per `(BoundedContext, AggregateType, AggregateID, AggregateVersion)`
- **Scalability**: Partition event store tables by bounded context for improved performance
- **Retention Policies**: Different contexts can have different data retention requirements

### Example Contexts

```go
// Identity context - user management
event := es.Event{
    BoundedContext: "Identity",
    AggregateType:  "User",
    AggregateID:    userID,
    EventType:      "UserCreated",
    // ...
}

// Billing context - subscription management
event := es.Event{
    BoundedContext: "Billing",
    AggregateType:  "Subscription",
    AggregateID:    subscriptionID,
    EventType:      "SubscriptionStarted",
    // ...
}

// Catalog context - product information
event := es.Event{
    BoundedContext: "Catalog",
    AggregateType:  "Product",
    AggregateID:    productID,
    EventType:      "ProductAdded",
    // ...
}
```

### Choosing Bounded Contexts

Bounded contexts should align with your business domains and organizational structure. Common examples:

- **Identity**: User accounts, authentication, profiles
- **Billing**: Subscriptions, payments, invoicing
- **Catalog**: Products, categories, inventory
- **Fulfillment**: Orders, shipping, tracking
- **Analytics**: Usage tracking, metrics, reporting

Each context should represent a distinct area of the business with its own terminology and rules.

## Core Components

### 1. Events

Events are immutable facts that have occurred in your system. They represent something that happened in the past and cannot be changed or deleted.

**Key principles:**

- Events are named in past tense: `UserCreated`, `OrderPlaced`, `PaymentProcessed`
- Events are immutable once persisted
- Events contain all data needed to understand what happened
- Events should be domain-focused, not technical

The Event struct represents an immutable domain event before it is stored. When you create events to append to the store, you populate this structure. The store then assigns the AggregateVersion and GlobalPosition when the event is persisted.

```go
type Event struct {
    CreatedAt      time.Time
    BoundedContext string          // Bounded context this event belongs to (e.g., "Identity", "Billing")
    AggregateType  string          // Type of aggregate (e.g., "User", "Order")
    EventType      string          // Type of event (e.g., "UserCreated", "OrderPlaced")
    AggregateID    string          // Aggregate instance identifier (UUID string, email, or any identifier)
    Payload        []byte          // Event data (typically JSON)
    Metadata       []byte          // Additional metadata (typically JSON)
    EventVersion   int             // Schema version of this event type (default: 1)
    CausationID    es.NullString  // ID of event/command that caused this event
    CorrelationID  es.NullString  // Link related events across aggregates
    TraceID        es.NullString  // Distributed tracing ID
    EventID        uuid.UUID       // Unique event identifier
}
```

Note that AggregateVersion and GlobalPosition are not part of the Event struct because they are assigned by the store during the Append operation. These fields are only present in PersistedEvent.

#### Event vs. PersistedEvent

**Event**: Used when creating new events to append to the store. You populate all fields except AggregateVersion and GlobalPosition, which the store assigns automatically during persistence.

**PersistedEvent**: Returned after events are stored or when reading from the store. Contains all Event fields plus the store-assigned GlobalPosition and AggregateVersion.

```go
type PersistedEvent struct {
    CreatedAt        time.Time
    BoundedContext   string
    AggregateType    string
    EventType        string
    AggregateID      string
    Payload          []byte
    Metadata         []byte
    GlobalPosition   int64     // Assigned by store - position in global event log
    AggregateVersion int64     // Assigned by store - version within this aggregate
    EventVersion     int
    CausationID      es.NullString
    CorrelationID    es.NullString
    TraceID          es.NullString
    EventID          uuid.UUID
}
```

This separation ensures events are value objects until persisted. The store assigns both position in the global log and version within the aggregate, guaranteeing consistency.

#### Event Design Best Practices

**✅ Good event names:**

- `OrderPlaced` (not `PlaceOrder` - it already happened)
- `PaymentCompleted` (not `Payment` - be specific)
- `UserEmailChanged` (not `UserUpdated` - what exactly changed?)

**❌ Bad event names:**

- `CreateUser` (command, not event)
- `Update` (too generic)
- `UserEvent` (meaningless)

**Event payload guidelines:**

- Include all data needed to understand the event
- Don't include computed values that can be derived
- Use JSON for flexibility and readability
- Version your event schemas (EventVersion field)

Example:
```go
// ✅ Good: Includes all relevant data
{
    "user_id": "123",
    "old_email": "alice@old.com",
    "new_email": "alice@new.com"
}

// ❌ Bad: Missing context
{
    "email": "alice@new.com"
}
```

### 2. Aggregates

An aggregate is a cluster of related domain objects that are treated as a unit for data changes. In event sourcing, an aggregate is the primary unit of consistency.

**Core principles:**

- An aggregate is a consistency boundary
- All events for an aggregate are processed in order
- Aggregates are identified by `AggregateType` + `AggregateID`
- Events within an aggregate are strictly ordered by `AggregateVersion`

**Example: User Aggregate**

```go
// User aggregate - spans multiple events
aggregateID := uuid.New().String()

events := []es.Event{
    {
        BoundedContext: "Identity",
        AggregateType:  "User",
        AggregateID:    aggregateID,
        EventType:      "UserCreated",
        EventVersion:   1,
        Payload:        []byte(`{"email":"alice@example.com"}`),
        Metadata:       []byte(`{}`),
        EventID:        uuid.New(),
        CreatedAt:      time.Now(),
    },
    {
        BoundedContext: "Identity",
        AggregateType:  "User",
        AggregateID:    aggregateID,  // Same aggregate
        EventType:      "EmailVerified",
        EventVersion:   1,
        Payload:        []byte(`{}`),
        Metadata:       []byte(`{}`),
        EventID:        uuid.New(),
        CreatedAt:      time.Now(),
    },
}
```

**Key principle:** All events for the same aggregate are processed in order.

### 3. Event Store

The event store is an append-only log of all events, providing atomic append operations with optimistic concurrency control.

```go
type EventStore interface {
    // Append events atomically with version control
    Append(ctx context.Context, tx es.DBTX, expectedVersion es.ExpectedVersion, events []es.Event) (es.AppendResult, error)
}
```

**Properties:**

- Append-only (events are never modified or deleted)
- Globally ordered (via `global_position`)
- Transactional (uses provided transaction)
- Optimistic concurrency via expectedVersion parameter

### 4. Consumers

Consumers process persisted events and produce side effects (read models, integration messages, audit updates, etc.).

A projection is a specific consumer that builds query-oriented read models.

**Two common consumer shapes:**

1. **Scoped Consumers** - Filter by aggregate type and/or bounded context:
```go
type ScopedConsumer interface {
    Consumer
    AggregateTypes() []string
    BoundedContexts() []string
}
```

2. **Global Consumers** - Receive all events:
```go
type Consumer interface {
    Name() string
    Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error
}
```

**Consumer lifecycle:**
1. Read batch of events from store starting from checkpoint
2. Apply partition filter (for horizontal scaling)
3. Apply aggregate/bounded-context filters (for scoped consumers)
4. Call Handle() for each event within a transaction
5. Update checkpoint atomically
6. Commit transaction
7. Repeat until context is cancelled or error occurs

### 5. Checkpoints

Checkpoints track where a consumer has processed up to.

```sql
CREATE TABLE consumer_checkpoints (
    consumer_name TEXT PRIMARY KEY,
    last_global_position BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL
);
```

**Key features:**

- One checkpoint per consumer
- Updated atomically with event processing
- Enables resumable processing

## Key Concepts

### Optimistic Concurrency

pupsourcing uses optimistic concurrency control to prevent conflicts.

```go
// Transaction 1
tx1, _ := db.BeginTx(ctx, nil)
store.Append(ctx, tx1, es.Exact(currentVersion), events1)  // Success
tx1.Commit()

// Transaction 2 (concurrent)
tx2, _ := db.BeginTx(ctx, nil)
store.Append(ctx, tx2, es.Exact(currentVersion), events2)  // ErrOptimisticConcurrency
tx2.Rollback()
```

**How it works:**
1. Each aggregate has a current version in `aggregate_heads` table
2. When appending, version is checked (O(1) lookup)
3. New events get consecutive versions
4. Database constraint enforces uniqueness: `(aggregate_type, aggregate_id, aggregate_version)`
5. If another transaction committed between check and insert → conflict

**Handling conflicts:**
```go
for retries := 0; retries < maxRetries; retries++ {
    tx, _ := db.BeginTx(ctx, nil)
    _, err := store.Append(ctx, tx, es.Exact(currentVersion), events)
    
    if errors.Is(err, store.ErrOptimisticConcurrency) {
        tx.Rollback()
        // Reload aggregate, reapply command
        continue
    }
    
    if err != nil {
        tx.Rollback()
        return err
    }
    
    return tx.Commit()
}
```

### Global Position

Every event gets a unique, monotonically increasing position.

```
Event 1 → global_position = 1
Event 2 → global_position = 2
Event 3 → global_position = 3
...
```

**Uses:**

- Checkpoint tracking
- Event replay
- Ordered processing
- Temporal queries

### Aggregate Versioning

Each aggregate has its own version sequence.

```
User ABC:
  Event 1 → aggregate_version = 1 (UserCreated)
  Event 2 → aggregate_version = 2 (EmailVerified)
  Event 3 → aggregate_version = 3 (NameChanged)

User XYZ:
  Event 1 → aggregate_version = 1 (UserCreated)
  Event 2 → aggregate_version = 2 (Deactivated)
```

**Uses:**

- Optimistic concurrency
- Event replay
- Aggregate reconstruction

### Idempotency

Projections must be idempotent because events may be reprocessed during crash recovery or restarts. This ensures that processing the same event multiple times produces the same result as processing it once.

**Non-idempotent (problematic):**
```go
func (p *Projection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    if event.EventType != "OrderPlaced" {
        return nil
    }
    
    // Problem: Running this twice increments the counter twice for the same event
    _, err := tx.ExecContext(ctx, 
        "UPDATE sales_statistics SET total_orders = total_orders + 1 WHERE date = CURRENT_DATE")
    return err
}
```

**Idempotent approach 1: Track processed events explicitly**
```go
func (p *Projection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    if event.EventType != "OrderPlaced" {
        return nil
    }
    
    // Check if we've already processed this event
    var exists bool
    err := tx.QueryRowContext(ctx,
        "SELECT EXISTS(SELECT 1 FROM order_events_processed WHERE event_id = $1)",
        event.EventID).Scan(&exists)
    if err != nil {
        return err
    }
    if exists {
        return nil  // Already processed, skip
    }
    
    // Process the event using the processor's transaction
    _, err = tx.ExecContext(ctx, 
        "UPDATE sales_statistics SET total_orders = total_orders + 1 WHERE date = CURRENT_DATE")
    if err != nil {
        return err
    }
    
    // Mark event as processed
    _, err = tx.ExecContext(ctx,
        "INSERT INTO order_events_processed (event_id, processed_at) VALUES ($1, NOW())",
        event.EventID)
    return err
}
```

**Idempotent approach 2: Use upsert semantics**

For type-safe event handling, use the [eventmap-gen](./eventmap-gen.md) tool to generate conversion functions:

```go
func (p *Projection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    if event.EventType != "UserCreated" {
        return nil
    }
    
    // Use generated code for type-safe conversion
    domainEvent, err := generated.FromESEvent(event)
    if err != nil {
        return err
    }
    
    userCreated, ok := domainEvent.(events.UserCreated)
    if !ok {
        return fmt.Errorf("unexpected event type")
    }
    
    // Use the processor's transaction for atomic updates
    _, err = tx.ExecContext(ctx,
        `INSERT INTO users (aggregate_id, email, name, created_at) 
         VALUES ($1, $2, $3, $4)
         ON CONFLICT (aggregate_id) 
         DO UPDATE SET email = EXCLUDED.email, name = EXCLUDED.name`,
        event.AggregateID, userCreated.Email, userCreated.Name, event.CreatedAt)
    return err
}
```

**Idempotent approach 3: Use event position as version**

```go
func (p *Projection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    if event.EventType != "InventoryAdjusted" {
        return nil
    }
    
    // Use generated code for type-safe conversion
    domainEvent, err := generated.FromESEvent(event)
    if err != nil {
        return err
    }
    
    inventoryAdjusted, ok := domainEvent.(events.InventoryAdjusted)
    if !ok {
        return fmt.Errorf("unexpected event type")
    }
    
    // Only apply if this event's position is greater than last processed position for this product
    result, err := tx.ExecContext(ctx,
        `UPDATE inventory 
         SET quantity = quantity + $1, last_event_position = $2
         WHERE product_id = $3 AND (last_event_position IS NULL OR last_event_position < $2)`,
        inventoryAdjusted.Quantity, event.GlobalPosition, inventoryAdjusted.ProductID)
    if err != nil {
        return err
    }
    
    // If no rows updated, product doesn't exist yet
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        _, err = tx.ExecContext(ctx,
            `INSERT INTO inventory (product_id, quantity, last_event_position) 
             VALUES ($1, $2, $3)
             ON CONFLICT (product_id) DO NOTHING`,  // Race condition safety
            payload.ProductID, payload.Quantity, event.GlobalPosition)
    }
    return err
}
```

### Transaction Boundaries

**You control transactions**, not the library.

```go
// Your responsibility: begin transaction
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

// Library uses your transaction
result, err := store.Append(ctx, tx, es.NoStream(), events)
if err != nil {
    return err  // Rollback happens in defer
}

// Your responsibility: commit
return tx.Commit()
```

**Benefits:**

- Compose operations atomically
- Control isolation levels
- Integrate with existing code

## Design Principles

### 1. Library, Not Framework

pupsourcing is designed as a library that you integrate into your application. Your code controls the flow and calls library functions when needed.

**Library approach (pupsourcing):**
```go
// You control when and how to run consumers
store := postgres.NewStore(postgres.DefaultStoreConfig())
w := postgres.NewWorker(db, store)
err := w.Run(ctx, &UserReadModel{})
```

**Framework approach (not pupsourcing):**
```go
// Framework discovers your code via reflection or annotations
@EventHandler
public void on(UserCreated event) { }
```

Benefits: You maintain full control over program flow, dependencies, and lifecycle management.

### 2. Explicit Dependencies

All dependencies are passed explicitly as parameters. There are no hidden globals, automatic dependency injection, or runtime discovery mechanisms.

**Explicit dependencies (pupsourcing):**
```go
// Every dependency is visible at the call site
store := postgres.NewStore(postgres.DefaultStoreConfig())
w := postgres.NewWorker(
    db,
    store,
    worker.WithTotalSegments(32),
)
err := w.Run(ctx, &UserReadModel{}, &OrderProjection{})
```

**Implicit dependencies (not pupsourcing):**
```go
// Where do consumers come from? Service locator? Global registry?
app.RunConsumers()
```

Benefits: Code is easier to understand, test, and debug when all dependencies are explicit.

### 3. Pull-Based Event Processing

Consumers actively read events from the store at their own pace. The library does not use publish-subscribe patterns or push-based delivery.

**Pull-based processing (pupsourcing):**
```go
// Consumer reads events on its own schedule
for {
    events := store.ReadEvents(ctx, tx, checkpoint, batchSize)
    for _, event := range events {
        consumer.Handle(ctx, tx, event)
    }
    // Update checkpoint and commit
}
```

Benefits:

- Natural backpressure mechanism (slow consumers won't overwhelm the system)
- No connection pooling or message broker management required
- Works consistently across different storage backends
- Simple failure recovery (resume from checkpoint)

### 4. Database-Centric Coordination

The database serves as the coordination mechanism. No external distributed systems are required for basic operation.

Database provides:

- **Checkpoints**: Each consumer tracks its position via a database row
- **Optimistic concurrency**: Enforced through unique constraints on aggregate versions
- **Transactional consistency**: Operations are atomic within database transactions

This approach keeps infrastructure requirements minimal while providing reliable coordination.

## Common Patterns

### Pattern 1: Read-Your-Writes

Write events and read them back within the same transaction for immediate consistency.

**Why this is useful:** 

- Enables strong consistency within a single operation
- Allows validation of command results before committing
- Useful for synchronous command handlers that need to return aggregate state
- Avoids eventual consistency delays when immediate feedback is required

**When to use this:**

- Command handlers that return the updated aggregate state
- Workflows requiring immediate validation of business rules
- APIs that need to return complete state after mutations
- Testing scenarios where you need to verify event results immediately

**Real-world example:**
In an e-commerce system, when a user places an order, you might want to return the order confirmation details immediately:

```go
// Command handler that returns order details
func PlaceOrder(ctx context.Context, db *sql.DB, store *postgres.Store, items []Item) (*Order, error) {
    tx, _ := db.BeginTx(ctx, nil)
    defer tx.Rollback()
    
    orderID := uuid.New().String()
    
    // Write order events
    events := []es.Event{
        {
            BoundedContext: "Orders",
            AggregateType:  "Order",
            AggregateID:    orderID,
            EventType:      "OrderCreated",
            EventVersion:   1,
            Payload:        marshalOrderCreated(items),
            Metadata:       []byte(`{}`),
            EventID:        uuid.New(),
            CreatedAt:      time.Now(),
        },
    }
    
    _, err := store.Append(ctx, tx, es.NoStream(), events)
    if err != nil {
        return nil, err
    }
    
    // Read back immediately in same transaction
    stream, err := store.ReadAggregateStream(ctx, tx, "Orders", "Order", orderID, nil, nil)
    if err != nil {
        return nil, err
    }
    
    // Reconstruct order from events
    order := reconstructOrder(stream.Events)
    
    // Commit and return complete order details
    if err := tx.Commit(); err != nil {
        return nil, err
    }
    
    return order, nil
}
```

### Pattern 2: Event Upcasting

Handle different event versions by converting older event formats to newer ones during processing.

**Why this is useful:**

- Enables schema evolution without data migration
- Maintains backward compatibility with historical events
- Allows business logic to work with a single current version
- Simplifies projection and aggregate reconstruction code

**When to use this:**

- Event schemas have evolved over time (added/removed/renamed fields)
- You want to normalize handling of different event versions
- Complex projections that would otherwise need version-specific logic
- Aggregate reconstruction should use consistent event structure

**Real-world example:**
A user registration event evolved to include additional fields. With [eventmap-gen](./eventmap-gen.md), the generated code handles versioning automatically:

```go
// Domain events (in your events/v1 and events/v2 directories)
// events/v1/user_registered.go
package v1

type UserRegistered struct {
    Email string
    Name  string
}

// events/v2/user_registered.go
package v2

type UserRegistered struct {
    Email   string
    Name    string
    Country string
}

// Use in projection - eventmap-gen handles version conversion
func (p *UserProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    if event.EventType == "UserRegistered" {
        // Use generated FromESEvent to get the right version
        domainEvent, err := generated.FromESEvent(event)
        if err != nil {
            return err
        }
        
        // Type switch on the domain event
        switch e := domainEvent.(type) {
        case v1.UserRegistered:
            // Handle v1 with default
            _, err = tx.ExecContext(ctx,
                "INSERT INTO users (aggregate_id, email, name, country) VALUES ($1, $2, $3, $4)",
                event.AggregateID, e.Email, e.Name, "Unknown")
            return err
            
        case v2.UserRegistered:
            // Handle v2 directly
            _, err = tx.ExecContext(ctx,
                "INSERT INTO users (aggregate_id, email, name, country) VALUES ($1, $2, $3, $4)",
                event.AggregateID, e.Email, e.Name, e.Country)
            return err
        }
    }
    return nil
}
```

### Pattern 3: Aggregate Reconstruction

Rebuild aggregate state from its event history.

**Domain Layer (Pure Business Logic):**

The domain aggregate should only work with domain events, not infrastructure types:

```go
// Domain event interface - all domain events implement this
type Event interface {
    isEvent() // Marker method
}

// Domain events (no infrastructure dependencies)
type UserCreated struct {
    Email string
    Name  string
}

func (UserCreated) isEvent() {}

type UserDeactivated struct {
    Reason string
}

func (UserDeactivated) isEvent() {}

type EmailChanged struct {
    NewEmail string
}

func (EmailChanged) isEvent() {}

// User aggregate in domain layer
type User struct {
    id     string
    email  string
    name   string
    active bool
}

// Apply accepts domain events only (not es.PersistedEvent)
func (u *User) Apply(event Event) {
    switch e := event.(type) {
    case UserCreated:
        u.email = e.Email
        u.name = e.Name
        u.active = true
    
    case UserDeactivated:
        u.active = false
    
    case EmailChanged:
        u.email = e.NewEmail
    }
}
```

**Infrastructure Layer (Persistence):**

The infrastructure layer converts persisted events to domain events:

```go
// Infrastructure code (uses eventmap-gen generated code)
import (
    "myapp/internal/domain/user"
    "myapp/internal/domain/user/events/v1"
    "myapp/internal/infrastructure/persistence/generated"
)

func LoadUser(ctx context.Context, tx es.DBTX, store store.AggregateStreamReader, id string) (*user.User, error) {
    // Read persisted events from store
    stream, err := store.ReadAggregateStream(ctx, tx, "Identity", "User", id, nil, nil)
    if err != nil {
        return nil, err
    }
    if stream.IsEmpty() {
        return nil, fmt.Errorf("user not found: %s", id)
    }
    
    // Convert infrastructure events to domain events using generated code
    // This keeps the domain layer pure
    domainEvents, err := generated.FromESEvents[user.Event](stream.Events)
    if err != nil {
        return nil, err
    }
    
    // Reconstitute aggregate from domain events
    u := user.NewUser(id)
    for _, event := range domainEvents {
        u.Apply(event)  // Works with domain events, not es.PersistedEvent
    }
    
    return u, nil
}
```

**Key principles:**

- **Domain layer**: Pure business logic, works only with domain events
- **Infrastructure layer**: Handles conversion between es.PersistedEvent and domain events  
- **No infrastructure creep**: Aggregate never depends on pupsourcing types
- **eventmap-gen helps**: Generated code handles conversion automatically
- **Type safety**: Use `FromESEvents[YourEventInterface]` with Go generics

## See Also

- [Getting Started](./getting-started.md) - Setup and first steps
- [Consumers](./consumers.md) - Consumer and projection patterns
- [Deployment](./deployment.md) - Running workers in production
- [Examples](https://github.com/pupsourcing/core/tree/master/examples) - Working code examples
