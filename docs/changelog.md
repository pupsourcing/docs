# Changelog

## v1.4.0 - 2026-02-22

### New Features

- **Dispatcher for projection workers** - Added optional `projection.Dispatcher` to reduce redundant idle polling across many projections in one process.

### Improvements

- **Recommended runtime pattern** - Use dispatcher + runner with shared `WakeupSource` for multi-projection workers.
- **Polling controls in `ProcessorConfig`** - Added `WakeupSource`, `PollInterval`, `MaxPollInterval`, `PollBackoffFactor`, and `WakeupJitter`.
- **Safety model remains unchanged** - Dispatcher is optimization-only; checkpointing and fallback polling still guarantee catch-up.

See [dispatcher-runner example](https://github.com/getpup/pupsourcing/tree/master/examples/dispatcher-runner) for lifecycle-safe integration.

## v0.0.3 - 2026-01-27

### Breaking Changes

⚠️ **Projection Handle Signature Change** - The `Projection.Handle` method now includes a transaction parameter to enable atomic read model and checkpoint updates.

**Before (v0.0.2):**
```go
func (p *MyProjection) Handle(ctx context.Context, event es.PersistedEvent) error
```

**After (v0.0.3):**
```go
func (p *MyProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error
```

**Migration Guide:**

All projections must update their Handle method signatures. The transaction parameter behavior:

**When Using the Same Database as Event Store:**
Use the `tx` parameter to execute database operations atomically with checkpoint updates:

```go
func (p *UserReadModel) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Use tx for database operations
    _, err := tx.ExecContext(ctx, 
        "INSERT INTO users (id, name) VALUES ($1, $2) "+
        "ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name",
        userID, name)
    return err
}
```

**When Writing to External Destinations:**
Ignore the `tx` parameter for projections writing to external systems (message brokers, separate databases, search engines):

```go
func (p *MessageBrokerPublisher) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Ignore tx - use message broker client
    _ = tx
    msg := message.NewMessage(event.EventID.String(), event.Payload)
    return p.publisher.Publish(event.EventType, msg)
}
```

**Benefits:**
- **Atomic Updates**: Read model changes and checkpoints are committed together
- **Data Consistency**: Eliminates the "projection succeeded but checkpoint failed" scenario
- **Simplified Code**: No need to manage separate transactions in SQL projections

### Improvements

- Transaction-based projection processing for better consistency
- All adapters (PostgreSQL, MySQL, SQLite) pass transaction to projections
- Processor manages transaction lifecycle automatically

## v0.0.2 - 2026-01-19

### New Features

- **RunModeOneOff for Projection Testing** - Added `RunModeOneOff` to enable synchronous projection processing for integration tests. Projections now support two modes:
  - `RunModeContinuous` (default) - Production mode that runs forever
  - `RunModeOneOff` - Testing mode that processes available events and exits cleanly
  
  This makes integration testing significantly easier and more deterministic. See the [One-Off Projection Processing guide](./projections.md#one-off-projection-processing) for details.

### Improvements

- All projection processors (postgres, mysql, sqlite) support the new run mode
- Comprehensive integration test examples added
- No breaking changes - fully backward compatible

## Previous Releases

See the [GitHub Releases page](https://github.com/getpup/pupsourcing/releases) for information about earlier versions.
