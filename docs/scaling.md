# Scaling

Guide to scaling projection processing across multiple workers.

## Table of Contents

1. [Horizontal Scaling](#horizontal-scaling)
2. [Partitioning Strategy](#partitioning-strategy)
3. [Running Multiple Projections](#running-multiple-projections)
4. [Performance Tuning](#performance-tuning)
5. [Production Patterns](#production-patterns)
6. [Advanced Topics](#advanced-topics)

## Horizontal Scaling

Scale projection processing by distributing work across multiple workers.

### When to Scale

**Indicators:**
- Projection lag increasing over time
- Event processing unable to keep pace with write rate
- Single worker resource constraints (CPU/memory)
- High-latency operations in projection handlers

**When Not to Scale:**
- Stable lag with adequate headroom
- Low event volume (< 100 events/second)
- Fast projection logic (< 1ms per event)
- Single worker not yet optimized (tune batch size first)

**Example Calculation:**

System generating 10,000 events/hour with per-event cost:
- Database insert: 2ms
- External API call: 50ms  
- Cache invalidation: 1ms
- Total: ~53ms per event

Throughput: ~18.9 events/second per worker
Required capacity: 2.78 events/second (10,000/hour)
Workers needed: 1 (with headroom)

For 100,000 events/hour: Need 2-3 workers minimum, use 4 for margin.

### Architecture

```
Event Stream → Partition Assignment → Worker Pool
  Event A   →   Partition 0      →   Worker 0
  Event B   →   Partition 1      →   Worker 1  
  Event C   →   Partition 2      →   Worker 2
  Event D   →   Partition 3      →   Worker 3
```

**Mechanism:**
1. Events assigned to partitions via `hash(aggregate_id)`
2. Each worker processes its assigned partition(s)
3. Workers operate independently without coordination
4. Single checkpoint per projection (not per partition)

### Hash-Based Partitioning

```go
config := projection.DefaultProcessorConfig()
config.PartitionKey = 0      // This worker (0-indexed)
config.TotalPartitions = 4   // Total worker count

processor := postgres.NewProcessor(db, store, &config)
```

**Characteristics:**
- Deterministic assignment: same aggregate → same partition
- Even distribution across workers
- No coordination overhead
- Maintains per-aggregate ordering

### Ordering Guarantees

✅ **Within Aggregate** - Events for same aggregate processed in order
✅ **Deterministic Assignment** - Same aggregate always routes to same partition
✅ **Even Distribution** - Approximately equal load per partition

❌ **Cross-Aggregate** - No global ordering between different aggregates

### Scaling Patterns

#### Pattern 1: Separate Processes

Run the same binary multiple times with different partition keys:

```bash
# Terminal 1
PARTITION_KEY=0 TOTAL_PARTITIONS=4 ./myapp process-projections

# Terminal 2
PARTITION_KEY=1 TOTAL_PARTITIONS=4 ./myapp process-projections

# Terminal 3
PARTITION_KEY=2 TOTAL_PARTITIONS=4 ./myapp process-projections

# Terminal 4
PARTITION_KEY=3 TOTAL_PARTITIONS=4 ./myapp process-projections
```

See [partitioned example](https://github.com/getpup/pupsourcing/tree/master/examples/partitioned) for details.

#### Pattern 2: Worker Pool (Single Process)

Run multiple partitions in the same process using goroutines:

```go
import "github.com/getpup/pupsourcing/es/projection/runner"

store := postgres.NewStore(postgres.DefaultStoreConfig())

// Create 4 processors with different partition keys
runners := make([]runner.ProjectionRunner, 4)
for i := 0; i < 4; i++ {
    config := projection.DefaultProcessorConfig()
    config.PartitionKey = i
    config.TotalPartitions = 4
    processor := postgres.NewProcessor(db, store, &config)
    runners[i] = runner.ProjectionRunner{
        Projection: projection,
        Processor:  processor,
    }
}

// Run all partitions concurrently
r := runner.New()
err := r.Run(ctx, runners)
```

**⚠️ Thread Safety Warning:** When using worker pools, all workers share the same projection instance. If your projection maintains state, it MUST be thread-safe:

```go
// ✅ Good: Thread-safe using atomic operations
type SafeProjection struct {
    count int64
}

func (p *SafeProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    atomic.AddInt64(&p.count, 1)  // Thread-safe
    return nil
}

// ✅ Good: Stateless (only updates database)
type StatelessProjection struct{}

func (p *StatelessProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    _, err := tx.ExecContext(ctx, "INSERT INTO read_model ...")  // Database handles concurrency
    return err
}

// ❌ Bad: Not thread-safe
type UnsafeProjection struct {
    count int  // Race condition!
}

func (p *UnsafeProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    p.count++  // NOT thread-safe!
    return nil
}
```

See [worker-pool example](https://github.com/getpup/pupsourcing/tree/master/examples/worker-pool) for details.

### When to Use Each Pattern

| Pattern | Best For | Pros | Cons |
|---------|----------|------|------|
| **Single Worker** | < 1K events/sec | Simple | Limited throughput |
| **Worker Pool** | 2-8 partitions, one machine | Easy deployment | Single point of failure |
| **Separate Processes** | > 8 partitions, multiple machines | Better isolation | More complex deployment |

## Partitioning Strategy

### Default: HashPartitionStrategy

Uses FNV-1a hash for deterministic, even distribution:

```go
type HashPartitionStrategy struct{}

func (HashPartitionStrategy) ShouldProcess(aggregateID string, partitionKey, totalPartitions int) bool {
    if totalPartitions <= 1 {
        return true
    }
    h := fnv.New32a()
    h.Write([]byte(aggregateID))
    partition := int(h.Sum32()) % totalPartitions
    return partition == partitionKey
}
```

### Custom Partitioning

Implement `PartitionStrategy` for custom logic:

```go
type RegionPartitionStrategy struct {
    region string
}

func (s RegionPartitionStrategy) ShouldProcess(aggregateID string, partitionKey, totalPartitions int) bool {
    // Custom logic - e.g., based on aggregate prefix
    // "us-" prefix goes to partition 0
    // "eu-" prefix goes to partition 1
    if strings.HasPrefix(aggregateID, "us-") {
        return partitionKey == 0
    } else if strings.HasPrefix(aggregateID, "eu-") {
        return partitionKey == 1
    }
    // Default to hash for others
    return HashPartitionStrategy{}.ShouldProcess(aggregateID, partitionKey, totalPartitions)
}
```

## Running Multiple Projections

### Pattern 1: Separate Processes

Run each projection in its own process for better isolation:

```bash
# Process 1
./myapp projection --name=user_counter

# Process 2
./myapp projection --name=analytics

# Process 3
./myapp projection --name=order_summary
```

### Pattern 2: Same Process

Run multiple projections in the same process:

```go
import "github.com/getpup/pupsourcing/es/projection/runner"

store := postgres.NewStore(postgres.DefaultStoreConfig())

// Recommended in v1.4.0+: run a shared dispatcher to reduce idle polling.
dispatcher := projection.NewDispatcher(db, store, nil)
go func() { _ = dispatcher.Run(ctx) }()

config := projection.DefaultProcessorConfig()
config.WakeupSource = dispatcher
config.PollInterval = 500 * time.Millisecond
config.MaxPollInterval = 8 * time.Second
config.PollBackoffFactor = 2.0
config.WakeupJitter = 25 * time.Millisecond

// A processor can still be reused for multiple projections.
processor := postgres.NewProcessor(db, store, &config)

r := runner.New()
err := r.Run(ctx, []runner.ProjectionRunner{
    {Projection: &UserCounterProjection{}, Processor: processor},
    {Projection: &EmailSenderProjection{}, Processor: processor},
    {Projection: &AnalyticsProjection{}, Processor: processor},
})
```

See [multiple-projections example](https://github.com/getpup/pupsourcing/tree/master/examples/multiple-projections) and [dispatcher-runner example](https://github.com/getpup/pupsourcing/tree/master/examples/dispatcher-runner) for details.

### Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| **Separate Processes** | Better isolation, independent scaling | More processes to manage |
| **Same Process** | Simpler deployment, shared resources | One failure affects all |

### When to Mix

You can run:
- Fast projections together
- Slow projections in separate processes with their own partitioning
- Critical projections isolated from non-critical ones

## Performance Tuning

### Batch Size

Controls how many events are processed per transaction:

```go
config := projection.DefaultProcessorConfig()
config.BatchSize = 100  // Default

// Larger batches: better throughput, higher latency
config.BatchSize = 1000

// Smaller batches: lower latency, more transactions
config.BatchSize = 10
```

**Guidelines:**
- Fast projections: 500-1000
- Slow projections: 10-50
- Default (100) works for most cases

### Connection Pooling

Configure database connection pool:

```go
db, _ := sql.Open("postgres", connStr)
db.SetMaxOpenConns(25)        // Limit concurrent connections
db.SetMaxIdleConns(5)         // Idle connections to keep
db.SetConnMaxLifetime(5 * time.Minute)
```

### Checkpoint Frequency

Checkpoint is updated after each batch. To reduce checkpoint writes:

```go
// Process more events per checkpoint
config.BatchSize = 500

// But consider: larger batches = more reprocessing on crash
```

### Poll Interval

Controls how frequently the processor checks for new events:

```go
config := projection.DefaultProcessorConfig()
config.PollInterval = 100 * time.Millisecond  // Default

// Reduce database load with longer intervals
config.PollInterval = 500 * time.Millisecond

// Lower latency with shorter intervals (more database queries)
config.PollInterval = 50 * time.Millisecond
```

**Trade-offs:**
- **Shorter intervals** (< 100ms): Lower projection latency, higher database load
- **Longer intervals** (> 100ms): Reduced database load, higher projection latency
- **Default (100ms)**: Balanced for most use cases

**When to adjust:**
- Increase for high database load scenarios
- Decrease for latency-sensitive projections
- Monitor database connections and adjust accordingly

### Wake-Up and Backoff Tuning (v1.4.0+)

With dispatcher-based workers, keep fallback polling enabled and tune it to reduce idle database load:

```go
config := projection.DefaultProcessorConfig()
config.WakeupSource = dispatcher       // shared projection.Dispatcher
config.PollInterval = 500 * time.Millisecond
config.MaxPollInterval = 8 * time.Second
config.PollBackoffFactor = 2.0
config.WakeupJitter = 25 * time.Millisecond
```

Defaults:
- `PollInterval`: `100ms`
- `MaxPollInterval`: `5s`
- `PollBackoffFactor`: `2.0`
- `WakeupJitter`: `25ms`
- `WakeupSource`: `nil`

### Monitoring

Track these metrics:

```sql
-- Projection lag
SELECT 
    projection_name,
    (SELECT MAX(global_position) FROM events) - last_global_position as lag
FROM projection_checkpoints;

-- Processing rate
SELECT 
    projection_name,
    last_global_position,
    updated_at
FROM projection_checkpoints
ORDER BY updated_at DESC;
```

## Production Patterns

### Pattern 1: Gradual Scaling

Start with 1 worker, scale up as needed:

```
Day 1: 1 worker (handles 100%)
Day 5: 2 workers (each handles ~50%)
Day 10: 4 workers (each handles ~25%)
Day 30: 8 workers (each handles ~12.5%)
```

See [scaling example](https://github.com/getpup/pupsourcing/tree/master/examples/scaling) for a demonstration.

### Pattern 2: Projection Prioritization

Run critical projections with more resources:

```go
store := postgres.NewStore(postgres.DefaultStoreConfig())

// Critical: User data (4 workers)
userRunners := make([]runner.ProjectionRunner, 4)
for i := 0; i < 4; i++ {
    config := projection.DefaultProcessorConfig()
    config.PartitionKey = i
    config.TotalPartitions = 4
    processor := postgres.NewProcessor(db, store, &config)
    userRunners[i] = runner.ProjectionRunner{
        Projection: &UserProjection{},
        Processor:  processor,
    }
}

// Normal: Analytics (2 workers, separate process)
analyticsRunners := make([]runner.ProjectionRunner, 2)
for i := 0; i < 2; i++ {
    config := projection.DefaultProcessorConfig()
    config.PartitionKey = i
    config.TotalPartitions = 2
    processor := postgres.NewProcessor(db, store, &config)
    analyticsRunners[i] = runner.ProjectionRunner{
        Projection: &AnalyticsProjection{},
        Processor:  processor,
    }
}

// Low priority: Reports (1 worker, best-effort)
config := projection.DefaultProcessorConfig()
processor := postgres.NewProcessor(db, store, &config)
processor.Run(ctx, &ReportProjection{})
```

### Pattern 3: Hot/Cold Separation

Process recent events quickly, older events more slowly:

```go
// Hot path: Recent events (small batches, low latency)
hotConfig := projection.DefaultProcessorConfig()
hotConfig.BatchSize = 10
hotProcessor := postgres.NewProcessor(db, store, &hotConfig)

// Cold path: Historical events (large batches, high throughput)
coldConfig := projection.DefaultProcessorConfig()
coldConfig.BatchSize = 1000
coldProcessor := postgres.NewProcessor(db, store, &coldConfig)
```

### Pattern 4: Idempotent Projections

Always make projections idempotent to handle reprocessing:

```go
// ✅ Good: Idempotent insert
_, err := tx.ExecContext(ctx,
    "INSERT INTO users (id, email) VALUES ($1, $2)"+
    "ON CONFLICT (id) DO UPDATE SET email = EXCLUDED.email",
    userID, email)

// ❌ Bad: Non-idempotent increment
_, err := tx.ExecContext(ctx,
    "UPDATE counters SET count = count + 1")

// ✅ Better: Track processed events
_, err := tx.ExecContext(ctx,
    "INSERT INTO processed_events (event_id) VALUES ($1)"+
    "ON CONFLICT (event_id) DO NOTHING",
    eventID)
```

## Advanced Topics

### Projection Rebuilding

To rebuild a projection from scratch:

```sql
-- 1. Delete checkpoint
DELETE FROM projection_checkpoints WHERE projection_name = 'my_projection';

-- 2. Clear read model
TRUNCATE TABLE my_read_model;

-- 3. Restart projection - it will reprocess all events
```

### Error Handling

```go
func (p *MyProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Transient errors: return error to retry
    // The processor will rollback the transaction automatically
    if err := someOperation(tx); err != nil {
        return fmt.Errorf("transient error: %w", err)
    }
    
    // Permanent errors: log and skip
    if err := validate(event); err != nil {
        log.Printf("Invalid event %s: %v", event.EventID, err)
        return nil  // Skip this event, transaction still commits
    }
    
    return nil
}
```

### Database Partitioning by Bounded Context

For high-volume systems with multiple bounded contexts, you can leverage PostgreSQL's table partitioning to improve query performance and data management. This is particularly useful when different contexts have different scaling characteristics or retention policies.

#### Why Partition by Bounded Context?

**Benefits:**
- **Improved Query Performance**: Queries filtering by bounded context only scan relevant partitions
- **Independent Scaling**: Different contexts can have different storage strategies
- **Flexible Retention**: Apply different retention policies per context (e.g., keep Identity events longer than Analytics)
- **Maintenance Efficiency**: Backup, restore, or vacuum individual contexts independently
- **Clear Data Boundaries**: Physical separation reinforces logical domain boundaries

**Example Scenario:**
- **Identity** context: 1M events/day, retain 7 years for compliance
- **Billing** context: 500K events/day, retain 10 years for audit
- **Analytics** context: 5M events/day, retain 90 days for operational insights

#### PostgreSQL List Partitioning by Bounded Context

Create partitions for each bounded context:

```sql
-- 1. Create partitioned events table (using your existing schema)
CREATE TABLE events (
    -- ... all your event columns here ...
    PRIMARY KEY (bounded_context, aggregate_type, aggregate_id, aggregate_version)
) PARTITION BY LIST (bounded_context);

-- 2. Create partition for Identity context
CREATE TABLE events_identity PARTITION OF events
    FOR VALUES IN ('Identity');

-- 3. Create partition for Billing context
CREATE TABLE events_billing PARTITION OF events
    FOR VALUES IN ('Billing');

-- 4. Create partition for Analytics context
CREATE TABLE events_analytics PARTITION OF events
    FOR VALUES IN ('Analytics');

-- Add indexes per partition for query performance
CREATE INDEX idx_events_identity_global_position ON events_identity (global_position);
CREATE INDEX idx_events_billing_global_position ON events_billing (global_position);
CREATE INDEX idx_events_analytics_global_position ON events_analytics (global_position);

-- 5. Partition aggregate_heads similarly
CREATE TABLE aggregate_heads (
    bounded_context TEXT NOT NULL,
    aggregate_type TEXT NOT NULL,
    aggregate_id TEXT NOT NULL,
    aggregate_version BIGINT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (bounded_context, aggregate_type, aggregate_id)
) PARTITION BY LIST (bounded_context);

CREATE TABLE aggregate_heads_identity PARTITION OF aggregate_heads
    FOR VALUES IN ('Identity');
CREATE TABLE aggregate_heads_billing PARTITION OF aggregate_heads
    FOR VALUES IN ('Billing');
CREATE TABLE aggregate_heads_analytics PARTITION OF aggregate_heads
    FOR VALUES IN ('Analytics');
```

**Query Performance Improvement:**
```sql
-- Without partitioning: Full table scan
SELECT * FROM events WHERE bounded_context = 'Identity' AND aggregate_type = 'User';

-- With partitioning: Only scans events_identity partition
SELECT * FROM events WHERE bounded_context = 'Identity' AND aggregate_type = 'User';
-- PostgreSQL automatically routes to events_identity partition
```

#### Adding New Bounded Contexts

When adding a new bounded context, create its partition:

```sql
-- Add new "Catalog" context
CREATE TABLE events_catalog PARTITION OF events
    FOR VALUES IN ('Catalog');

CREATE INDEX idx_events_catalog_global_position 
    ON events_catalog (global_position);
CREATE INDEX idx_events_catalog_aggregate 
    ON events_catalog (aggregate_type, aggregate_id);
CREATE INDEX idx_events_catalog_created_at 
    ON events_catalog (created_at);

CREATE TABLE aggregate_heads_catalog PARTITION OF aggregate_heads
    FOR VALUES IN ('Catalog');
```

**⚠️ Important:** You must create a partition before writing events to that bounded context, otherwise PostgreSQL will reject the insert.

#### Hash Partitioning with Subpartitions

For contexts with extremely high volume, combine bounded context partitioning with hash-based subpartitioning:

```sql
-- Events table partitioned by bounded context
CREATE TABLE events (
    -- columns as before
    PRIMARY KEY (bounded_context, aggregate_type, aggregate_id, aggregate_version)
) PARTITION BY LIST (bounded_context);

-- Identity partition with hash subpartitions on aggregate_type
CREATE TABLE events_identity PARTITION OF events
    FOR VALUES IN ('Identity')
    PARTITION BY HASH (aggregate_type);

CREATE TABLE events_identity_0 PARTITION OF events_identity
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE events_identity_1 PARTITION OF events_identity
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE events_identity_2 PARTITION OF events_identity
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE events_identity_3 PARTITION OF events_identity
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Billing partition without subpartitioning (lower volume)
CREATE TABLE events_billing PARTITION OF events
    FOR VALUES IN ('Billing');
```

**When to use subpartitions:**
- Query latency on individual bounded context partitions becomes unacceptable for your use case
- Write throughput to a single context partition approaches database limits
- Multiple high-volume aggregate types within the same context would benefit from physical separation
- Different aggregate types have significantly different access patterns or retention needs

#### Multi-Column Hash Partitioning

For even distribution across multiple dimensions:

```sql
-- Composite hash on (bounded_context, aggregate_type, aggregate_id)
CREATE TABLE events (
    -- columns as before
) PARTITION BY HASH (bounded_context, aggregate_type, aggregate_id);

-- Create 16 partitions for even distribution
CREATE TABLE events_00 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE events_01 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 1);
CREATE TABLE events_02 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 2);
CREATE TABLE events_03 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 3);
CREATE TABLE events_04 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 4);
CREATE TABLE events_05 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 5);
CREATE TABLE events_06 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 6);
CREATE TABLE events_07 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 7);
CREATE TABLE events_08 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 8);
CREATE TABLE events_09 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 9);
CREATE TABLE events_10 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 10);
CREATE TABLE events_11 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 11);
CREATE TABLE events_12 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 12);
CREATE TABLE events_13 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 13);
CREATE TABLE events_14 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 14);
CREATE TABLE events_15 PARTITION OF events FOR VALUES WITH (MODULUS 16, REMAINDER 15);
```

**Benefit:** Automatic load balancing across partitions without managing per-context partitions.

**Trade-off:** Lose ability to manage contexts independently (e.g., different retention policies).

#### Migration Strategy

To migrate from non-partitioned to partitioned tables:

```sql
-- 1. Create new partitioned table
CREATE TABLE events_partitioned (
    -- same columns as events
) PARTITION BY LIST (bounded_context);

-- 2. Create partitions for each context
-- (as shown above)

-- 3. Copy data in batches
INSERT INTO events_partitioned
SELECT * FROM events
WHERE bounded_context = 'Identity'
LIMIT 100000;
-- Repeat until all Identity events migrated
-- Then repeat for other contexts

-- 4. Swap tables (in transaction)
BEGIN;
ALTER TABLE events RENAME TO events_old;
ALTER TABLE events_partitioned RENAME TO events;
COMMIT;

-- 5. Verify and drop old table
DROP TABLE events_old;
```

#### Partition Maintenance

**Detach old partitions for archival:**
```sql
-- Detach Analytics partition for archival (after 90 days)
ALTER TABLE events DETACH PARTITION events_analytics;

-- Archive to cold storage
pg_dump -t events_analytics > analytics_archive.sql

-- Drop or keep as standalone table
DROP TABLE events_analytics;
```

**Attach archived partition for historical queries:**
```sql
-- Restore from archive
psql -f analytics_archive.sql

-- Reattach
ALTER TABLE events ATTACH PARTITION events_analytics
    FOR VALUES IN ('Analytics');
```

#### Application Code Considerations

**No changes required in pupsourcing code** - partitioning is transparent:

```go
// Same code works with or without partitioning
events := []es.Event{
    {
        BoundedContext: "Identity",  // Routes to events_identity partition
        AggregateType:  "User",
        AggregateID:    userID,
        // ...
    },
}

result, err := store.Append(ctx, tx, es.NoStream(), events)
```

**For ScopedProjection filtering:**
```go
// Projection automatically benefits from partition pruning
func (p *UserReadModel) BoundedContexts() []string {
    return []string{"Identity"}  // PostgreSQL only scans events_identity
}

func (p *UserReadModel) AggregateTypes() []string {
    return []string{"User"}
}
```

#### Monitoring Partition Performance

```sql
-- Check partition sizes
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE tablename LIKE 'events_%'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Verify partition pruning in queries
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM events
WHERE bounded_context = 'Identity' AND aggregate_type = 'User';
-- Should show "Partitions removed: N" in output
```

#### Best Practices

1. **Plan contexts upfront**: Adding partitions is easy, but reorganizing is expensive
2. **Create partitions before use**: Inserts fail if partition doesn't exist
3. **Index each partition separately**: Indexes are not inherited automatically
4. **Monitor partition sizes**: Rebalance if one partition grows disproportionately
5. **Use consistent naming**: `events_{context_name_lower}` for clarity
6. **Document retention policies**: Make context-specific rules explicit
7. **Test migrations**: Always test partition changes on staging first

#### When NOT to Partition

- **Few contexts (<3)**: Overhead outweighs benefits
- **Low volume (<10M events total)**: Partitioning adds complexity without gains
- **Uniform access patterns**: If all queries scan all contexts, partitioning doesn't help
- **Small team**: Operational complexity may not be worth it

## See Also

- [Getting Started](./getting-started.md) - Basic setup
- [Scaling Example](https://github.com/getpup/pupsourcing/tree/master/examples/scaling) - Dynamic scaling demonstration
- [Deployment Guide](./deployment.md) - Production deployment patterns
