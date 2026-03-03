# Event Mapping Code Generation

The `eventmap-gen` tool generates strongly-typed mapping code between domain events and pupsourcing event sourcing types (`es.Event` and `es.PersistedEvent`).

## Table of Contents

- [Why This Tool Exists](#why-this-tool-exists)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Versioned Events](#versioned-events)
- [Generated Code](#generated-code)
- [Clean Architecture](#clean-architecture)
- [Advanced Usage](#advanced-usage)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Why This Tool Exists

In event-sourced systems, you typically have:

1. **Domain events** - Pure business logic types in your domain layer (no infrastructure dependencies)
2. **ES events** - Infrastructure types (`es.Event`, `es.PersistedEvent`) for storage and replay

Manually mapping between these layers is error-prone and repetitive. This tool:

- **Generates explicit mapping code** - No runtime reflection, no magic
- **Supports versioned events** - Handle schema evolution over time
- **Maintains clean architecture** - Domain stays pure, generated code lives in infrastructure
- **Type-safe** - Compile-time guarantees with generics
- **Inspectable** - Generated code is readable Go that you can debug

## Installation

**Go 1.24+** (recommended): Use the new `go get -tool` feature to install tools locally to your module:

```bash
go get -tool github.com/pupsourcing/core/cmd/eventmap-gen@latest
```

This allows you to use it with `go generate` directives:

```go
//go:generate go tool eventmap-gen -input ../../domain/events -output . -package persistence
```

**Go 1.23 and earlier**: Use `go run` directly in your `go:generate` directives (no installation needed):

```go
//go:generate go run github.com/pupsourcing/core/cmd/eventmap-gen -input ../../domain/events -output . -package persistence
```

Or run the tool directly from the command line:

```bash
go run github.com/pupsourcing/core/cmd/eventmap-gen [flags]
```

## Quick Start

### 1. Organize Your Domain Events

Create domain event structs in a directory:

```
internal/domain/events/
  v1/
    user_registered.go
    user_email_changed.go
```

**Example domain event (`user_registered.go`):**

```go
package v1

// UserRegistered is emitted when a new user registers.
type UserRegistered struct {
    Email string `json:"email"`
    Name  string `json:"name"`
}
```

### 2. Generate Mapping Code

The recommended approach is using `go generate` with a directive in your repository adapter:

```bash
go generate ./...
```

This runs any `//go:generate` directives in your code. See [Using with go generate](#using-with-go-generate) below for setup.

Alternatively, run the tool directly:

```bash
go run github.com/pupsourcing/core/cmd/eventmap-gen \
  -input internal/domain/events \
  -output internal/infrastructure/persistence/generated \
  -package generated
```

Or if you installed the tool:

```bash
eventmap-gen \
  -input internal/domain/events \
  -output internal/infrastructure/persistence/generated \
  -package generated
```

### 3. Use Generated Code

```go
package main

import (
    "github.com/pupsourcing/core/es"
    "github.com/google/uuid"
    "internal/domain/events/v1"
    "internal/infrastructure/persistence/generated"
)

func main() {
    // Create domain event
    event := v1.UserRegistered{
        Email: "alice@example.com",
        Name:  "Alice",
    }

    // Convert to es.Event
    esEvents, err := generated.ToESEvents(
        "Identity",                // bounded context
        "User",                    // aggregate type
        uuid.New().String(),       // aggregate ID
        []v1.UserRegistered{event},// domain events (type-safe!)
        generated.WithTraceID("trace-123"), // optional metadata
    )

    // Store in event store...
    // Later, retrieve and convert back...

    persistedEvents := []es.PersistedEvent{/* from database */}
    domainEvents, err := generated.FromESEvents[any](persistedEvents)
}
```

## Versioned Events

Event schemas evolve over time. This tool supports versioning through directory structure, similar to protobuf packages.

### Directory Structure

```
events/
  v1/
    user_registered.go    # Initial version
    order_created.go
  v2/
    user_registered.go    # New version with additional fields
    order_created.go      # New version
  v3/
    order_created.go      # Another evolution
```

### Version Rules

1. **Directory name determines version**: `v1/` → `EventVersion = 1`, `v2/` → `EventVersion = 2`
2. **Event type stays the same**: `UserRegistered` is the event type across all versions
3. **Version + Type uniquely identifies schema**: `(UserRegistered, 1)` vs `(UserRegistered, 2)`
4. **Default version is 1**: Events outside version directories get version 1

### Example: Schema Evolution

**Version 1 (`events/v1/user_registered.go`):**

```go
package v1

type UserRegistered struct {
    Email string `json:"email"`
    Name  string `json:"name"`
}
```

**Version 2 (`events/v2/user_registered.go`):**

```go
package v2

type UserRegistered struct {
    Email     string `json:"email"`
    Name      string `json:"name"`
    Country   string `json:"country"`   // New field
    Timestamp int64  `json:"timestamp"` // New field
}
```

### Handling Historical Events

The generated code correctly deserializes events based on their stored version:

```go
// Event stream from database contains mixed versions
persistedEvents := []es.PersistedEvent{
    {EventType: "UserRegistered", EventVersion: 1, Payload: ...},
    {EventType: "UserEmailChanged", EventVersion: 1, Payload: ...},
    {EventType: "UserRegistered", EventVersion: 2, Payload: ...},
}

// Deserialize correctly based on version
domainEvents, err := generated.FromESEvents[any](persistedEvents)
// Result:
// - domainEvents[0] is v1.UserRegistered
// - domainEvents[1] is v1.UserEmailChanged
// - domainEvents[2] is v2.UserRegistered
```

## Generated Code

The tool generates:

### 1. `EventTypeOf` - Type Resolution

```go
func EventTypeOf(e any) (string, error)
```

Returns the event type string for a domain event. The event type is the struct name (without version).

### 2. `ToESEvents` - Domain to ES Conversion

```go
func ToESEvents[T any](
    boundedContext string,
    aggregateType string,
    aggregateID string,
    events []T,
    opts ...Option,
) ([]es.Event, error)
```

Converts domain events to `es.Event` instances with:

- JSON marshaling of payload
- Automatic version assignment
- UUID generation
- Optional metadata (causation/correlation/trace IDs)

The generic type parameter `T` allows for type-safe event slices. You can pass `[]YourEventType` instead of `[]any` for better type safety. Using `[]any` is still supported for convenience when mixing event types.

### 3. `FromESEvents` - ES to Domain Conversion (Batch)

```go
func FromESEvents[T any](events []es.PersistedEvent) ([]T, error)
```

Converts persisted events back to domain events using generics. Validates event type and version, then unmarshals JSON payload.

### 4. `FromESEvent` - ES to Domain Conversion (Single)

```go
func FromESEvent(pe es.PersistedEvent) (any, error)
```

Converts a single persisted event to a domain event. This is particularly useful in projection handlers where you process events one at a time:

```go
func (p *MyProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Convert the persisted event to a domain event
    domainEvent, err := generated.FromESEvent(event)
    if err != nil {
        return fmt.Errorf("failed to convert event: %w", err)
    }
    
    // Handle the specific event type with the processor's transaction
    switch e := domainEvent.(type) {
    case v1.UserRegistered:
        return p.handleUserRegistered(ctx, tx, e)
    case v1.UserEmailChanged:
        return p.handleUserEmailChanged(ctx, tx, e)
    default:
        return nil // Ignore unknown events
    }
}
```

### 5. Type-Safe Helpers

For each event version, generates:

```go
// Convert specific event to ES
func ToUserRegisteredV1(
    boundedContext string,
    aggregateType string,
    aggregateID string,
    e v1.UserRegistered,
    opts ...Option,
) (es.Event, error)

// Convert ES to specific event
func FromUserRegisteredV1(
    pe es.PersistedEvent,
) (v1.UserRegistered, error)
```

### 6. Observability and Metadata Options

```go
type Option func(*eventOptions)

func WithCausationID(id string) Option
func WithCorrelationID(id string) Option
func WithTraceID(id string) Option
func WithMetadata(metadata []byte) Option
```

Use options to inject metadata:

```go
esEvents, err := generated.ToESEvents(
    "Identity", "User", userID, []v1.UserRegistered{event},
    generated.WithCausationID("command-123"),
    generated.WithCorrelationID("correlation-456"),
    generated.WithTraceID("trace-789"),
)
```

### 7. Unit Tests

The tool automatically generates comprehensive unit tests in a separate `_test.go` file. The generated tests include:

- **EventTypeOf tests** - Verify correct event type resolution for all events
- **ToESEvents tests** - Test type-safe conversion from domain to ES events
- **FromESEvents tests** - Test conversion from persisted events back to domain events
- **Options tests** - Verify metadata injection via options pattern
- **Type helpers tests** - Test version-specific conversion functions
- **Error cases** - Test handling of unknown event types and invalid JSON

These tests ensure the generated code works correctly and provide examples of usage patterns.

## Clean Architecture

This tool maintains clean architecture boundaries:

```
┌─────────────────────────────────────────┐
│ Domain Layer (Pure Business Logic)      │
│ ┌─────────────────────────────────────┐ │
│ │ Domain Events (NO dependencies)     │ │
│ │ - v1.UserRegistered                 │ │
│ │ - v1.OrderCreated                   │ │
│ │ - v2.UserRegistered                 │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
               ▲
               │ Pure, no framework coupling
               │
┌──────────────┴──────────────────────────┐
│ Infrastructure Layer                    │
│ ┌─────────────────────────────────────┐ │
│ │ Generated Mapping Code              │ │
│ │ (Depends on pupsourcing & domain)   │ │
│ │ - EventTypeOf()                     │ │
│ │ - ToESEvents()                      │ │
│ │ - FromESEvents()                    │ │
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ Event Store (PostgreSQL)             │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Design Principles

1. **Domain events are pure** - No dependency on pupsourcing or infrastructure
2. **Generated code lives in infrastructure** - Can depend on pupsourcing and domain
3. **No Apply logic** - Generated code only handles marshaling/unmarshaling
4. **No aggregate modification** - Aggregate logic stays in domain layer
5. **Explicit, not magical** - Generated code is readable Go

### DDD / CQRS / Event Sourcing

This tool fits into Domain-Driven Design and CQRS patterns:

- **Events are Facts** - Domain events represent things that happened
- **Immutable** - Events never change after creation
- **Version-Aware** - Schema evolution is explicit and traceable
- **Command-Event Separation** - Commands produce events, events are stored
- **Read Models** - Projections consume events to build read models

#### Repository Adapter Example

Here's how to integrate the generator in a repository adapter using `go generate`:

**Directory structure:**
```
internal/
  domain/
    user/
      events/
        v1/
          user_events.go       # Pure domain events
      user.go                  # Aggregate
      repository.go            # Repository interface (port)
  infrastructure/
    persistence/
      user/
        repository.go          # Repository adapter (implementation)
```

**Repository adapter with go generate directive:**

```go
// internal/infrastructure/persistence/user/repository.go

//go:generate go tool eventmap-gen -input ../../../domain/user/events -output . -package user

package user

import (
    "context"
    "database/sql"

    "github.com/pupsourcing/core/es"
    "github.com/pupsourcing/core/es/adapters/postgres"
    
    "myapp/internal/domain/user"
    "myapp/internal/domain/user/events/v1"
)

// Repository implements the domain repository interface using event sourcing.
type Repository struct {
    db    *sql.DB
    store interface {
        store.EventStore
        store.AggregateStreamReader
    }
}

func NewRepository(db *sql.DB) *Repository {
    return &Repository{
        db:    db,
        store: postgres.NewStore(postgres.DefaultStoreConfig()),
    }
}

func (r *Repository) Save(ctx context.Context, u *user.User) error {
    // Get uncommitted domain events from aggregate
    domainEvents := u.GetUncommittedEvents()
    
    // Convert to ES events using generated code
    esEvents, err := ToESEvents(u.BoundedContext(), u.AggregateType(), u.ID(), domainEvents)
    if err != nil {
        return err
    }
    
    tx, err := r.db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Get expected version from aggregate
    expectedVersion := u.Version()
    
    // Append to event store with optimistic concurrency
    result, err := r.store.Append(ctx, tx, es.Exact(expectedVersion), esEvents)
    if err != nil {
        return err
    }
    
    // Update aggregate version after successful append
    // This is crucial for optimistic concurrency control
    u.SetVersion(result.ToVersion())
    
    return tx.Commit()
}

func (r *Repository) Load(ctx context.Context, id string) (*user.User, error) {
    // Read aggregate stream
    stream, err := r.store.ReadAggregateStream(
        ctx, r.db, "User", id, nil, nil,
    )
    if err != nil {
        return nil, err
    }
    
    // Convert to domain events using generated code
    domainEvents, err := FromESEvents[any](stream.Events)
    if err != nil {
        return nil, err
    }
    
    // Reconstitute aggregate from events with correct version
    u := user.FromEvents(id, domainEvents)
    u.SetVersion(stream.Version())  // Set the current version from stream
    
    return u, nil
}
```

**Generate the mapping code:**

```bash
# From project root
go generate ./internal/infrastructure/persistence/user/...
```

This creates `event_mapping.gen.go` and `event_mapping.gen_test.go` in the same directory as the repository, keeping all infrastructure code together.

**Benefits of this approach:**

- Code generation is co-located with the adapter that uses it
- Running `go generate ./...` regenerates all mapping code
- CI/CD can verify generated code is up to date with `go generate -x ./... && git diff --exit-code`
- Clear separation between domain (pure events) and infrastructure (persistence)

## Advanced Usage

### Custom Module Paths

By default, the tool auto-detects your module path from `go.mod`. Override with:

```bash
eventmap-gen \
  -input internal/domain/events \
  -output internal/infra/generated \
  -package generated \
  -module github.com/mycompany/myapp/internal/domain/events
```

### Custom Output Filename

```bash
eventmap-gen \
  -input events \
  -output generated \
  -filename my_events.gen.go
```

### Using with `go generate`

The recommended approach is to add `//go:generate` directives in your infrastructure code.

**For Go 1.24+**, after installing with `go get -tool`:

```go
//go:generate go tool eventmap-gen -input ../../domain/events -output . -package persistence
```

**Why use `go tool`?**
- Uses the module's local tool installation (via `go get -tool`)
- No need to specify full import path
- Faster execution (no recompilation)
- Version pinned to your module's requirements

**For Go 1.23 and earlier**, use `go run`:

```go
//go:generate go run github.com/pupsourcing/core/cmd/eventmap-gen -input ../../domain/events -output . -package persistence
```

Then run:

```bash
go generate ./...
```

See the [Repository Adapter Example](#repository-adapter-example) for a complete integration pattern.

## Best Practices

### 1. Keep Domain Events Simple

✅ **Good:**

```go
package v1

type OrderCreated struct {
    OrderID    string  `json:"order_id"`
    CustomerID string  `json:"customer_id"`
    Amount     float64 `json:"amount"`
}
```

❌ **Avoid:**

```go
package v1

import "github.com/pupsourcing/core/es"

// Don't couple domain to infrastructure
type OrderCreated struct {
    es.Event  // Don't embed ES types
    Amount float64
}
```

### 2. Version When Schema Changes

Create a new version when:

- Adding required fields
- Changing field types
- Removing fields
- Changing field semantics

Optional fields can sometimes be added to existing versions using `omitempty`.

### 3. Don't Delete Old Versions

Old versions must remain for replaying historical events:

```
events/
  v1/
    user_registered.go  # Keep this even if you move to v2
  v2/
    user_registered.go  # New version
```

### 4. Prefer Type-Safe Generic Calls

The generic `ToESEvents` function now supports type-safe slices:

```go
// ✅ Best: Type-safe with generics (compile-time safety)
events := []v1.UserRegistered{event1, event2}
esEvents, err := generated.ToESEvents("Identity", "User", userID, events)

// ✅ Good: Type-safe per-event helper
esEvent, err := generated.ToUserRegisteredV1("Identity", "User", userID, event)

// ⚠️ OK but less safe: Using []any for mixed event types
esEvents, err := generated.ToESEvents("Identity", "User", userID, []any{event1, event2})
```

Using type-safe slices with generics provides better compile-time safety while maintaining flexibility.

### 5. Document Breaking Changes

Add comments when introducing breaking schema changes:

```go
package v2

// UserRegistered version 2 adds Country (required) and Timestamp.
// Breaking change from v1: Country is now required.
// Use v1.UserRegistered for historical events before 2024-01-15.
type UserRegistered struct {
    Email     string `json:"email"`
    Name      string `json:"name"`
    Country   string `json:"country"`   // New required field
    Timestamp int64  `json:"timestamp"` // Added in v2
}
```

## Troubleshooting

### Error: "no events discovered"

**Cause:** No exported structs found in input directory.

**Solution:** Ensure:

- Structs are exported (capitalized names)
- Files are `.go` files (not `_test.go`)
- Directory path is correct

### Error: "unknown event type"

**Cause:** Trying to deserialize an event type that wasn't in the input directory when code was generated.

**Solution:** 
1. Add the event type to your domain events directory
2. Regenerate the mapping code
3. Redeploy

### Error: "unknown version X for event type Y"

**Cause:** Event store contains a version that wasn't in the input directory when code was generated.

**Solution:**
1. Add the missing version to your domain events directory (e.g., create `vX/event.go`)
2. Regenerate the mapping code
3. Redeploy

### Import Path Issues

If you see import errors in generated code:

1. Check that `-module` flag is set correctly
2. Verify `go.mod` is in the expected location
3. Run `go mod tidy` after generating code

### Generated Code Won't Compile

1. Ensure domain events have exported fields
2. Check for circular imports
3. Verify all types are JSON-serializable
4. Run `go mod tidy` to resolve dependencies

## Examples

### Example 1: User Management Events

```
events/
  v1/
    user_registered.go
    user_email_changed.go
    user_deleted.go
```

```bash
eventmap-gen \
  -input events \
  -output persistence/generated
```

### Example 2: Order Processing with Versioning

```
events/
  v1/
    order_created.go
    order_shipped.go
  v2/
    order_created.go  # Added tax and currency
```

```bash
eventmap-gen \
  -input events \
  -output persistence/generated
```

### Example 3: Multi-Aggregate System

```
events/
  user/
    v1/
      user_registered.go
  order/
    v1/
      order_created.go
      order_fulfilled.go
    v2/
      order_created.go
```

Generate separately for each aggregate:

```bash
eventmap-gen -input events/user -output persistence/user/generated
eventmap-gen -input events/order -output persistence/order/generated
```

## Further Reading

- [Event Sourcing Pattern](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Versioned Domain Events](https://www.eventstore.com/blog/versioning-in-an-event-sourced-system)
- [pupsourcing Core Concepts](core-concepts.md)
- [pupsourcing Consumers Guide](consumers.md)
