# API Reference

Complete API documentation for pupsourcing.

## Table of Contents

1. [Core Types](#core-types)
2. [Observability](#observability)
3. [Event Store](#event-store)
4. [Projections](#projections)
5. [Runner Package](#runner-package)
6. [PostgreSQL Adapter](#postgresql-adapter)

## Core Types

### es.Event

Represents an immutable domain event before persistence.

```go
type Event struct {
    CreatedAt      time.Time
    BoundedContext string          // Bounded context this event belongs to (e.g., "Identity", "Billing")
    AggregateType  string          // Type of aggregate (e.g., "User", "Order")
    EventType      string          // Type of event (e.g., "UserCreated")
    AggregateID    string          // Aggregate instance identifier (UUID string, email, or any identifier)
    Payload        []byte          // Event data (typically JSON)
    Metadata       []byte          // Additional metadata (typically JSON)
    EventVersion   int             // Schema version of this event type (default: 1)
    CausationID    es.NullString  // ID of event/command that caused this event
    CorrelationID  es.NullString  // Links related events across aggregates
    TraceID        es.NullString  // Distributed tracing ID
    EventID        uuid.UUID       // Unique event identifier
}
```

**Note:** `AggregateVersion` and `GlobalPosition` are assigned by the store during `Append`. The field order is optimized for memory layout.

### es.ExpectedVersion

Controls optimistic concurrency for aggregate updates.

```go
type ExpectedVersion struct {
    // internal value
}

// Constructors
func Any() ExpectedVersion         // No version check
func NoStream() ExpectedVersion    // Aggregate must not exist
func Exact(version int64) ExpectedVersion  // Aggregate must be at specific version
```

**Usage:**

- **Any()**: Skip version validation. Use when you don't need concurrency control.
- **NoStream()**: Enforce that the aggregate doesn't exist. Use for aggregate creation and uniqueness enforcement.
- **Exact(N)**: Enforce that the aggregate is at version N. Use for normal command handling with optimistic concurrency.

**Examples:**

```go
// Creating a new aggregate
_, err := store.Append(ctx, tx, es.NoStream(), []es.Event{event})

// Updating an existing aggregate at version 5
_, err := store.Append(ctx, tx, es.Exact(5), []es.Event{event})

// No concurrency check
_, err := store.Append(ctx, tx, es.Any(), []es.Event{event})

// Uniqueness enforcement via reservation aggregate
email := "user@example.com"
reservationEvent := es.Event{
    BoundedContext: "Identity",
    AggregateType:  "EmailReservation",
    AggregateID:    email,  // Use email as aggregate ID
    // ... other fields
}
_, err := store.Append(ctx, tx, es.NoStream(), []es.Event{reservationEvent})
// Second attempt with same email will fail with ErrOptimisticConcurrency
```

### es.PersistedEvent

Represents an event that has been stored, including position information.

```go
type PersistedEvent struct {
    CreatedAt        time.Time
    BoundedContext   string
    AggregateType    string
    EventType        string
    AggregateID      string
    Payload          []byte
    Metadata         []byte
    GlobalPosition   int64       // Position in global event log (assigned by store)
    AggregateVersion int64       // Version of aggregate after this event (assigned by store)
    EventVersion     int
    CausationID      es.NullString
    CorrelationID    es.NullString
    TraceID          es.NullString
    EventID          uuid.UUID
}
```

**Note:** The field order is optimized for memory layout.

### es.DBTX

Database transaction interface used throughout the library.

```go
type DBTX interface {
    ExecContext(ctx context.Context, query string, args ...interface{}) (sql.Result, error)
    QueryContext(ctx context.Context, query string, args ...interface{}) (*sql.Rows, error)
    QueryRowContext(ctx context.Context, query string, args ...interface{}) *sql.Row
}
```

Implemented by both `*sql.DB` and `*sql.Tx`.

## Observability

### es.Logger

Optional logging interface for instrumenting the library.

```go
type Logger interface {
    Debug(ctx context.Context, msg string, keyvals ...interface{})
    Info(ctx context.Context, msg string, keyvals ...interface{})
    Error(ctx context.Context, msg string, keyvals ...interface{})
}
```

**Usage:**
```go
type MyLogger struct {
    logger *slog.Logger
}

func (l *MyLogger) Debug(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.DebugContext(ctx, msg, keyvals...)
}

func (l *MyLogger) Info(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.InfoContext(ctx, msg, keyvals...)
}

func (l *MyLogger) Error(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.ErrorContext(ctx, msg, keyvals...)
}

// Inject into store
config := postgres.DefaultStoreConfig()
config.Logger = &MyLogger{logger: slog.Default()}
store := postgres.NewStore(config)
```

See the [Observability Guide](./observability.md) for complete documentation and examples.

### es.NoOpLogger

Default logger implementation that does nothing. Used internally when no logger is configured.

```go
type NoOpLogger struct{}

func (NoOpLogger) Debug(_ context.Context, _ string, _ ...interface{}) {}
func (NoOpLogger) Info(_ context.Context, _ string, _ ...interface{}) {}
func (NoOpLogger) Error(_ context.Context, _ string, _ ...interface{}) {}
```

## Event Store

### store.EventStore

Interface for appending events.

```go
type EventStore interface {
    Append(ctx context.Context, tx es.DBTX, expectedVersion es.ExpectedVersion, events []es.Event) (es.AppendResult, error)
}
```

#### Append

Atomically appends events within a transaction with optimistic concurrency control.

```go
func (s *EventStore) Append(ctx context.Context, tx es.DBTX, expectedVersion es.ExpectedVersion, events []es.Event) (es.AppendResult, error)
```

**Parameters:**
- `ctx`: Context for cancellation
- `tx`: Database transaction (you control transaction boundaries)
- `expectedVersion`: Expected aggregate version (Any, NoStream, or Exact)
- `events`: Events to append (must all be for the same aggregate)

**Returns:**
- `es.AppendResult`: Result containing persisted events and global positions
  - `Events`: The persisted events with assigned versions
  - `GlobalPositions`: Assigned global positions for the events
  - Helper methods: `FromVersion()`, `ToVersion()`
- `error`: Error if any (including `ErrOptimisticConcurrency`)

**Errors:**
- `store.ErrOptimisticConcurrency`: Version conflict or expectation mismatch
- `store.ErrNoEvents`: Empty events slice

**Example:**
```go
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

// Create a new aggregate
result, err := store.Append(ctx, tx, es.NoStream(), events)
if errors.Is(err, store.ErrOptimisticConcurrency) {
    // Aggregate already exists
}

fmt.Printf("Aggregate version: %d\n", result.ToVersion())
fmt.Printf("Positions: %v\n", result.GlobalPositions)

// Update existing aggregate at version 3
result, err := store.Append(ctx, tx, es.Exact(3), events)
if errors.Is(err, store.ErrOptimisticConcurrency) {
    // Version mismatch - another transaction updated the aggregate
    // Reload aggregate state and retry
}

tx.Commit()
```

### store.EventReader

Interface for reading events sequentially.

```go
type EventReader interface {
    ReadEvents(ctx context.Context, tx es.DBTX, fromPosition int64, limit int) ([]es.PersistedEvent, error)
}
```

#### ReadEvents

Reads events starting from a position.

```go
func (s *EventReader) ReadEvents(ctx context.Context, tx es.DBTX, fromPosition int64, limit int) ([]es.PersistedEvent, error)
```

**Parameters:**
- `ctx`: Context
- `tx`: Database transaction
- `fromPosition`: Start position (exclusive - returns events AFTER this position)
- `limit`: Maximum number of events to return

**Returns:**
- `[]es.PersistedEvent`: Events ordered by global_position
- `error`: Error if any

### store.AggregateStreamReader

Interface for reading events for a specific aggregate.

```go
type AggregateStreamReader interface {
    ReadAggregateStream(ctx context.Context, tx es.DBTX, aggregateType string, 
                       aggregateID string, fromVersion, toVersion *int64) (es.Stream, error)
}
```

#### ReadAggregateStream

Reads all events for an aggregate, optionally filtered by version range.

```go
func (s *Store) ReadAggregateStream(ctx context.Context, tx es.DBTX, 
                                   boundedContext, aggregateType string, aggregateID string,
                                   fromVersion, toVersion *int64) (es.Stream, error)
```

**Parameters:**
- `boundedContext`: Bounded context of the aggregate (e.g., "Identity", "Billing")
- `aggregateType`: Type of aggregate (e.g., "User")
- `aggregateID`: Aggregate instance ID (string: UUID, email, or any identifier)
- `fromVersion`: Optional minimum version (inclusive). Pass `nil` for all.
- `toVersion`: Optional maximum version (inclusive). Pass `nil` for all.

**Returns:**
- `es.Stream`: Stream containing:
  - `AggregateType`: The aggregate type
  - `AggregateID`: The aggregate ID
  - `Events`: Events ordered by aggregate_version
  - Helper methods: `Version()`, `IsEmpty()`, `Len()`
- `error`: Error if any

**Examples:**
```go
// Read all events for UUID-based aggregate
userID := uuid.New().String()
stream, _ := store.ReadAggregateStream(ctx, tx, "Identity", "User", userID, nil, nil)
fmt.Printf("Aggregate version: %d\n", stream.Version())
fmt.Printf("Event count: %d\n", stream.Len())

// Process events
for _, event := range stream.Events {
    // Handle event
}

// Read from version 5 onwards
from := int64(5)
stream, _ := store.ReadAggregateStream(ctx, tx, "Identity", "User", userID, &from, nil)

// Read specific range
to := int64(10)
stream, _ := store.ReadAggregateStream(ctx, tx, "Identity", "User", userID, &from, &to)

// Read reservation aggregate by email
stream, _ := store.ReadAggregateStream(ctx, tx, "Identity", "EmailReservation", "user@example.com", nil, nil)

// Check if aggregate exists
if stream.IsEmpty() {
    // Aggregate has no events
}
```

## Projections

Projections transform events into query-optimized read models. There are two types:

1. **Scoped Projections** - Filter events by aggregate type (for read models)
2. **Global Projections** - Receive all events (for integration publishers, audit logs)

### projection.ScopedProjection

Optional interface for projections that only need specific aggregate types.

```go
type ScopedProjection interface {
    Projection
    AggregateTypes() []string
}
```

#### When to Use

**Use ScopedProjection for:**
- Read models for specific aggregates (e.g., user profile view)
- Domain-specific denormalizations (e.g., order summary)
- Search indexes for specific entity types

**Use Projection (global) for:**
- Message broker integrations (Watermill, Kafka, RabbitMQ)
- Outbox pattern implementations
- Complete audit trails
- Cross-aggregate analytics

#### AggregateTypes

Returns the list of aggregate types this projection processes.

```go
func (p *UserReadModelProjection) AggregateTypes() []string {
    return []string{"User"}  // Only receives User events
}

func (p *OrderUserProjection) AggregateTypes() []string {
    return []string{"User", "Order"}  // Receives User and Order events
}
```

**Behavior:**
- If list is non-empty, only events matching these aggregate types are delivered to `Handle()`
- If list is empty, projection receives all events (same as global projection)
- Filtering happens at processor level, not in handler (O(1) map lookup)

#### Example: Scoped Read Model

```go
type UserReadModelProjection struct {
    db *sql.DB
}

func (p *UserReadModelProjection) Name() string {
    return "user_read_model"
}

func (p *UserReadModelProjection) AggregateTypes() []string {
    return []string{"User"}
}

func (p *UserReadModelProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Only User events arrive here
    // Use processor's transaction for atomic updates
    switch event.EventType {
    case "UserCreated":
        _, err := tx.ExecContext(ctx, "INSERT INTO users_read_model ...")
        return err
    case "UserUpdated":
        _, err := tx.ExecContext(ctx, "UPDATE users_read_model ...")
        return err
    }
    return nil
}
```

### projection.Projection

Interface for event projection handlers. Projections are storage-agnostic and can write to any destination (SQL, NoSQL, message brokers, search engines, etc.).

```go
type Projection interface {
    Name() string
    Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error
}
```

**Breaking Change (v1.2.0):** Added `tx *sql.Tx` parameter to `Handle()` method to enable atomic read model and checkpoint updates.

#### Example: Global Integration Publisher

```go
type WatermillPublisher struct {
    publisher message.Publisher
}

func (p *WatermillPublisher) Name() string {
    return "system.integration.watermill.v1"
}

// No AggregateTypes() method - receives ALL events

func (p *WatermillPublisher) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Ignore tx for non-SQL projections
    _ = tx
    
    // Use message broker client
    msg := message.NewMessage(event.EventID.String(), event.Payload)
    return p.publisher.Publish(event.EventType, msg)
}
```

#### Example: SQL Read Model Projection

```go
type UserReadModelProjection struct {
    // No need to store db connection - use tx parameter
}

func (p *UserReadModelProjection) Name() string {
    return "user_read_model"
}

func (p *UserReadModelProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Use processor's transaction for atomic updates
    if event.EventType == "UserCreated" {
        _, err := tx.ExecContext(ctx, 
            "INSERT INTO users_read_model (id, name) VALUES ($1, $2) "+
            "ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name",
            event.AggregateID, name)
        return err
    }
    return nil
}
```

#### Name

Returns unique projection name used for checkpoint tracking.

```go
func (p *MyProjection) Name() string {
    return "my_projection"
}
```

#### Handle

Processes a single event. The processor provides its transaction to enable atomic updates.

```go
func (p *MyProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Process event
    return nil
}
```

**Parameters:**
- `ctx`: Context for cancellation
- `tx`: Processor's transaction for atomic read model updates (use for SQL, ignore for non-SQL)
- `event`: Event to process (passed by value to enforce immutability)

**Returns:**
- `error`: Return error to stop projection processing and rollback transaction

**Important:** 
- Make projections idempotent - events may be reprocessed on crash recovery
- When using the same database as event store: Use `tx` for all database operations
- When writing to external destinations: Ignore `tx` and manage your own connections
- Never call `tx.Commit()` or `tx.Rollback()` - processor manages transaction lifecycle
- Return an error to trigger rollback of both read model changes and checkpoint update

### postgres.Processor

PostgreSQL-specific processor for running projections. Manages SQL transactions and checkpointing internally.

**Breaking Change (v1.2.0):** Moved from `projection.Processor` to adapter-specific implementations. Each adapter (postgres, mysql, sqlite) has its own processor.

#### NewProcessor

Creates a new PostgreSQL projection processor.

```go
func NewProcessor(db *sql.DB, store *postgres.Store, config *projection.ProcessorConfig) *Processor
```

**Parameters:**
- `db`: PostgreSQL database connection
- `store`: PostgreSQL store (implements EventReader and CheckpointStore)
- `config`: Processor configuration (passed by pointer)

**Returns:**
- `*Processor`: Processor instance

**Example:**
```go
store := postgres.NewStore(postgres.DefaultStoreConfig())
config := projection.DefaultProcessorConfig()
processor := postgres.NewProcessor(db, store, &config)
```

### projection.Dispatcher

Optional coordinator for projection workers running in the same process.

It centralizes idle polling and emits best-effort wake-up signals to processors. This reduces redundant empty polling queries when many projections are running together.

**Correctness model:** Dispatcher is optimization-only. Delivery guarantees still come from checkpoints + processor pull logic + fallback polling.

#### DefaultDispatcherConfig

Returns default dispatcher configuration.

```go
cfg := projection.DefaultDispatcherConfig()
cfg.PollInterval = 150 * time.Millisecond // Optional override

dispatcher := projection.NewDispatcher(db, store, &cfg)
err := dispatcher.Run(ctx)
```

#### Integration with runner

```go
dispatcher := projection.NewDispatcher(db, store, nil)

cfg := projection.DefaultProcessorConfig()
cfg.WakeupSource = dispatcher

processor := postgres.NewProcessor(db, store, &cfg)
```

### projection.Processor (Legacy)

**Deprecated in v1.2.0** - Use adapter-specific processors instead:

Processes events for a projection.

```go
type Processor struct {
    // unexported fields
}
```

#### NewProcessor

Creates a new projection processor.

```go
func NewProcessor(txProvider TxProvider, eventReader store.EventReader, checkpointStore store.CheckpointStore, config *ProcessorConfig) *Processor
```

**Parameters:**
- `txProvider`: Transaction provider (typically `*sql.DB`)
- `eventReader`: Event reader implementation
- `checkpointStore`: Checkpoint store implementation
- `config`: Processor configuration (passed by pointer)

**Breaking Change (v1.2.0):** Added `checkpointStore` parameter to decouple from database-specific checkpoint logic. The `txProvider` replaces the previous `db` parameter to support non-SQL implementations.

**Breaking Change (v1.1.0):** Changed from `ProcessorConfig` (value) to `*ProcessorConfig` (pointer) for better performance.

**Returns:**
- `*Processor`: Processor instance

#### Run

Runs the projection until context is cancelled or an error occurs.

```go
func (p *Processor) Run(ctx context.Context, projection Projection) error
```

**Parameters:**
- `ctx`: Context for cancellation
- `projection`: Projection to run

**Returns:**
- `error`: Error if projection handler fails, or `ctx.Err()` on cancellation

**Example:**
```go
store := postgres.NewStore(postgres.DefaultStoreConfig())
config := projection.DefaultProcessorConfig()
processor := postgres.NewProcessor(db, store, &config)

ctx, cancel := context.WithCancel(context.Background())
defer cancel()

err := processor.Run(ctx, myProjection)
if errors.Is(err, context.Canceled) {
    log.Println("Projection stopped gracefully")
}
```

### projection.ProcessorConfig

Configuration for projection processor.

```go
type ProcessorConfig struct {
    PartitionStrategy PartitionStrategy  // Partitioning strategy
    WakeupSource      WakeupSource       // Optional wake-up source (dispatcher)
    Logger            es.Logger          // Optional logger (nil = disabled)
    RunMode           RunMode            // Processing mode (continuous or one-off)
    BatchSize         int                // Events per batch
    PartitionKey      int                // This worker's partition (0-indexed)
    TotalPartitions   int                // Total number of partitions
    PollInterval      time.Duration      // Base fallback polling interval
    MaxPollInterval   time.Duration      // Cap for polling backoff
    PollBackoffFactor float64            // Polling backoff multiplier
    WakeupJitter      time.Duration      // Jitter before wake-up handling
}
```

**Note:** Fields are ordered by size (interfaces/pointers first) for optimal memory layout.

#### RunMode

Determines processing behavior:
- `RunModeContinuous` (default): Runs forever, continuously polling for new events
- `RunModeOneOff`: Processes all available events and exits cleanly

Default: `RunModeContinuous`

Use `RunModeOneOff` for integration tests and one-time catch-up operations.

**Breaking Change (v1.2.0):** Removed `EventsTable` and `CheckpointsTable` fields. Table configuration is now handled by the adapter's store configuration.

#### Wake-Up and Polling Fields (v1.4.0+)

- `WakeupSource`: Optional signal source (typically `projection.Dispatcher`)
- `PollInterval`: Base fallback polling interval
- `MaxPollInterval`: Maximum fallback polling interval during backoff
- `PollBackoffFactor`: Multiplier used when backing off idle polling
- `WakeupJitter`: Jitter before wake-up processing to smooth spikes

#### DefaultProcessorConfig

Returns default configuration.

```go
func DefaultProcessorConfig() ProcessorConfig {
    return ProcessorConfig{
        BatchSize:         100,
        PartitionKey:      0,
        TotalPartitions:   1,
        PollInterval:      100 * time.Millisecond,
        MaxPollInterval:   5 * time.Second,
        PollBackoffFactor: 2.0,
        WakeupJitter:      25 * time.Millisecond,
        WakeupSource:      nil,
        PartitionStrategy: HashPartitionStrategy{},
        Logger:            nil,  // No logging by default
        RunMode:           RunModeContinuous,  // Continuous mode by default
    }
}
```

### projection.RunMode

Determines how the projection processor handles event processing.

**Type:** `int`

**Constants:**
- `RunModeContinuous` (0) - Default. Runs forever, continuously polling for new events. Use for production.
- `RunModeOneOff` (1) - Processes all available events and exits cleanly. Use for testing and catch-up operations.

**Example:**
```go
config := projection.DefaultProcessorConfig()
config.RunMode = projection.RunModeOneOff  // Use one-off mode for tests

processor := postgres.NewProcessor(db, store, &config)
err := processor.Run(ctx, proj)  // Exits when all events are processed
```

### projection.PartitionStrategy

Interface for partitioning strategies.

```go
type PartitionStrategy interface {
    ShouldProcess(aggregateID string, partitionKey, totalPartitions int) bool
}
```

### projection.HashPartitionStrategy

Default hash-based partitioning strategy.

```go
type HashPartitionStrategy struct{}

func (HashPartitionStrategy) ShouldProcess(aggregateID string, partitionKey, totalPartitions int) bool
```

Uses FNV-1a hash for deterministic, even distribution.

## Runner Package

### runner.Runner

Orchestrates multiple projections.

```go
type Runner struct {
    // unexported fields
}
```

#### New

Creates a new runner.

```go
func New(db *sql.DB, eventReader store.EventReader) *Runner
```

#### Run

Runs multiple projections concurrently.

```go
func (r *Runner) Run(ctx context.Context, configs []ProjectionConfig) error
```

**Parameters:**
- `ctx`: Context for cancellation
- `configs`: Projection configurations

**Returns:**
- `error`: First error from any projection, or `ctx.Err()`

**Example:**
```go
store := postgres.NewStore(postgres.DefaultStoreConfig())
runner := runner.New()

// Create processors for each projection
config1 := projection.DefaultProcessorConfig()
processor1 := postgres.NewProcessor(db, store, &config1)

config2 := projection.DefaultProcessorConfig()
processor2 := postgres.NewProcessor(db, store, &config2)

// Run projections
runners := []runner.ProjectionRunner{
    {Projection: proj1, Processor: processor1},
    {Projection: proj2, Processor: processor2},
}

err := runner.Run(ctx, runners)
```

### runner.ProjectionRunner

Pairs a projection with its adapter-specific processor.

```go
type ProjectionRunner struct {
    Projection projection.Projection
    Processor  projection.ProcessorRunner
}
```

**Fields:**
- `Projection`: The projection to run
- `Processor`: Adapter-specific processor (postgres.Processor, mysql.Processor, etc.)

**Example:**
```go
store := postgres.NewStore(postgres.DefaultStoreConfig())
config := projection.DefaultProcessorConfig()
config.PartitionKey = 0
config.TotalPartitions = 4
processor := postgres.NewProcessor(db, store, &config)

runner := runner.New()
err := runner.Run(ctx, []runner.ProjectionRunner{
    {Projection: &MyProjection{}, Processor: processor},
})
```

## PostgreSQL Adapter

### postgres.Store

PostgreSQL implementation of EventStore, EventReader, AggregateStreamReader, and CheckpointStore.

```go
type Store struct {
    // unexported fields
}
```

#### NewStore

Creates a new PostgreSQL store.

```go
func NewStore(config StoreConfig) *Store
```

**Parameters:**
- `config`: Store configuration

**Returns:**
- `*Store`: Store instance

**Example:**
```go
store := postgres.NewStore(postgres.DefaultStoreConfig())
```

### postgres.StoreConfig

Configuration for PostgreSQL store.

```go
type StoreConfig struct {
    Logger              es.Logger  // Optional logger (nil = disabled)
    EventsTable         string     // Events table name
    AggregateHeadsTable string     // Aggregate heads table name
    CheckpointsTable    string     // Checkpoints table name
}
```

**Note:** Fields are ordered by size (interfaces/pointers first) for optimal memory layout.

#### DefaultStoreConfig

Returns default configuration.

```go
func DefaultStoreConfig() StoreConfig {
    return StoreConfig{
        EventsTable:         "events",
        AggregateHeadsTable: "aggregate_heads",
        CheckpointsTable:    "projection_checkpoints",
        Logger:              nil,  // No logging by default
    }
}
```

## Error Types

### store.ErrOptimisticConcurrency

Returned when a version conflict is detected.

```go
var ErrOptimisticConcurrency = errors.New("optimistic concurrency conflict")
```

**Example:**
```go
_, err := store.Append(ctx, tx, es.Exact(currentVersion), events)
if errors.Is(err, store.ErrOptimisticConcurrency) {
    // Retry transaction
}
```

### store.ErrNoEvents

Returned when attempting to append zero events.

```go
var ErrNoEvents = errors.New("no events to append")
```

### projection.ErrProjectionStopped

Returned when a projection stops due to handler error.

```go
var ErrProjectionStopped = errors.New("projection stopped")
```

### runner.ErrNoProjections

Returned when no projections are provided.

```go
var ErrNoProjections = errors.New("no projections provided")
```

### runner.ErrInvalidPartitionConfig

Returned when partition configuration is invalid.

```go
var ErrInvalidPartitionConfig = errors.New("invalid partition configuration")
```

## See Also

- [Getting Started](./getting-started.md) - Setup and basic usage
- [Core Concepts](./core-concepts.md) - Understanding the architecture
- [Scaling Guide](./scaling.md) - Production patterns
- [Examples](../examples/) - Working code examples
