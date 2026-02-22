# Deployment & Operations Guide

This guide covers production deployment patterns, monitoring, and operational best practices for pupsourcing.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Deployment Patterns](#deployment-patterns)
- [Configuration Management](#configuration-management)
- [Monitoring](#monitoring)
- [Operational Best Practices](#operational-best-practices)
- [Troubleshooting](#troubleshooting)
- [See Also](#see-also)

## Prerequisites

Before deploying, you need to implement a `run-projections` command in your application. This command should:

1. Parse configuration from environment variables or flags
2. Initialize database connection
3. Create projection instances
4. Use the runner package to start processing (and use a shared dispatcher for multi-projection workers)

**Example implementation:**

```go
package main

import (
    "context"
    "database/sql"
    "flag"
    "log"
    "os"
    "os/signal"
    "strconv"
    "syscall"
    
    "github.com/getpup/pupsourcing/es/adapters/postgres"
    "github.com/getpup/pupsourcing/es/projection"
    "github.com/getpup/pupsourcing/es/projection/runner"
    _ "github.com/lib/pq"
)

func main() {
    // Parse flags
    partitionKey := flag.Int("partition-key", -1, "Partition key")
    totalPartitions := flag.Int("total-partitions", 1, "Total partitions")
    flag.Parse()
    
    // Also support environment variables
    if *partitionKey == -1 {
        if envKey := os.Getenv("PARTITION_KEY"); envKey != "" {
            *partitionKey, _ = strconv.Atoi(envKey)
        } else {
            *partitionKey = 0
        }
    }
    if envTotal := os.Getenv("TOTAL_PARTITIONS"); envTotal != "" {
        *totalPartitions, _ = strconv.Atoi(envTotal)
    }
    
    // Connect to database
    dbURL := os.Getenv("DATABASE_URL")
    db, err := sql.Open("postgres", dbURL)
    if err != nil {
        log.Fatalf("Failed to connect to database: %v", err)
    }
    defer db.Close()
    
    // Create store
    store := postgres.NewStore(postgres.DefaultStoreConfig())
    
    // Configure projection
    config := projection.DefaultProcessorConfig()
    config.PartitionKey = *partitionKey
    config.TotalPartitions = *totalPartitions
    
    // Create your projections
    userProjection := &UserReadModelProjection{db: db}
    
    // Run with context cancellation
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Handle shutdown signals
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    go func() {
        <-sigChan
        log.Println("Shutdown signal received")
        cancel()
    }()
    
    // Start processing
    processor := postgres.NewProcessor(db, store, &config)
    if err := processor.Run(ctx, userProjection); err != nil {
        log.Fatalf("Projection failed: %v", err)
    }
}
```

### Recommended for Multi-Projection Workers (v1.4.0+)

When one process runs multiple continuous projections, prefer `projection.Dispatcher` + `runner` wiring so processors share wake-up signals:

```go
runCtx, cancel := context.WithCancel(ctx)
defer cancel()

dispatcher := projection.NewDispatcher(db, store, nil)
dispatcherErrCh := make(chan error, 1)
go func() { dispatcherErrCh <- dispatcher.Run(runCtx) }()

newProcessor := func() *postgres.Processor {
    cfg := projection.DefaultProcessorConfig()
    cfg.WakeupSource = dispatcher
    cfg.PollInterval = 500 * time.Millisecond
    cfg.MaxPollInterval = 8 * time.Second
    cfg.PollBackoffFactor = 2.0
    cfg.WakeupJitter = 25 * time.Millisecond
    return postgres.NewProcessor(db, store, &cfg)
}

runners := []runner.ProjectionRunner{
    {Projection: &UserReadModelProjection{db: db}, Processor: newProcessor()},
    {Projection: &AnalyticsProjection{db: db}, Processor: newProcessor()},
}

runErr := runner.New().Run(runCtx, runners)
cancel()
dispatcherErr := <-dispatcherErrCh

if runErr != nil && !errors.Is(runErr, context.Canceled) {
    return runErr
}
if dispatcherErr != nil &&
    !errors.Is(dispatcherErr, context.Canceled) &&
    !errors.Is(dispatcherErr, context.DeadlineExceeded) {
    // warn only: dispatcher is optimization-only
}
```

This reduces empty polling queries and database pressure while preserving correctness guarantees via checkpointing + fallback polling.

For multiple projections or partitioned execution, see the [scaling guide](./scaling.md) and [examples](https://github.com/getpup/pupsourcing/tree/master/examples).

## Deployment Patterns

Choosing the right deployment pattern depends on your scale, infrastructure, and operational requirements.

### Pattern 1: Single Binary, Multiple Instances

**When to use:**
- Need horizontal scaling with partitioning
- Running multiple projection workers
- Simple infrastructure (Docker Compose, VMs, cloud instances)
- Want explicit control over worker configuration

**Pros:**
- Simple to understand and debug
- Explicit configuration
- Easy to scale incrementally
- Works anywhere (Docker, VMs, cloud, on-premise)

**Cons:**
- Manual configuration for each instance
- More deployment units to manage
- Need to track partition assignments

**Best for:** Small to medium deployments, development, teams comfortable with Docker Compose or systemd.

Run the same binary multiple times with different configuration.

#### Docker Compose

**Use case:** Local development, staging environments, or small production deployments.

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  projection-worker-0:
    image: myapp:latest
    depends_on:
      - postgres
    environment:
      DATABASE_URL: "postgres://postgres:postgres@postgres:5432/myapp?sslmode=disable"
      PARTITION_KEY: "0"
      TOTAL_PARTITIONS: "4"
    command: ["./myapp", "run-projections"]
    restart: unless-stopped

  projection-worker-1:
    image: myapp:latest
    depends_on:
      - postgres
    environment:
      DATABASE_URL: "postgres://postgres:postgres@postgres:5432/myapp?sslmode=disable"
      PARTITION_KEY: "1"
      TOTAL_PARTITIONS: "4"
    command: ["./myapp", "run-projections"]
    restart: unless-stopped

  projection-worker-2:
    image: myapp:latest
    depends_on:
      - postgres
    environment:
      DATABASE_URL: "postgres://postgres:postgres@postgres:5432/myapp?sslmode=disable"
      PARTITION_KEY: "2"
      TOTAL_PARTITIONS: "4"
    command: ["./myapp", "run-projections"]
    restart: unless-stopped

  projection-worker-3:
    image: myapp:latest
    depends_on:
      - postgres
    environment:
      DATABASE_URL: "postgres://postgres:postgres@postgres:5432/myapp?sslmode=disable"
      PARTITION_KEY: "3"
      TOTAL_PARTITIONS: "4"
    command: ["./myapp", "run-projections"]
    restart: unless-stopped

volumes:
  postgres_data:
```

#### Kubernetes

**Deployment Approach:** pupsourcing projections with partitioning require **StatefulSets**, not regular Deployments, because each worker needs a stable, predictable partition key.

##### Option 1: StatefulSet (Recommended for Partitioned Projections)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: projection-workers
spec:
  serviceName: projection-workers
  replicas: 4
  selector:
    matchLabels:
      app: projection-workers
  template:
    metadata:
      labels:
        app: projection-workers
    spec:
      initContainers:
      - name: set-partition-key
        image: busybox
        command:
        - sh
        - -c
        - |
          # Extract ordinal from hostname (projection-workers-0, projection-workers-1, etc.)
          ORDINAL=$(hostname | grep -o '[0-9]*$')
          echo "PARTITION_KEY=$ORDINAL" > /config/partition.env
        volumeMounts:
        - name: config
          mountPath: /config
      containers:
      - name: worker
        image: myapp:latest
        command: ["sh", "-c", "source /config/partition.env && ./myapp run-projections"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: TOTAL_PARTITIONS
          value: "4"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: projection-workers
spec:
  clusterIP: None  # Headless service for StatefulSet
  selector:
    app: projection-workers
  ports:
  - port: 8080
    targetPort: 8080
```

##### Option 2: Regular Deployment (Only for Non-Partitioned Projections)

If you're running projections **without partitioning** (i.e., `TotalPartitions=1`), you can use a regular Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: projection-worker
spec:
  replicas: 1  # Only 1 replica for non-partitioned
  selector:
    matchLabels:
      app: projection-worker
  template:
    metadata:
      labels:
        app: projection-worker
    spec:
      containers:
      - name: worker
        image: myapp:latest
        command: ["./myapp", "run-projections"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: PARTITION_KEY
          value: "0"
        - name: TOTAL_PARTITIONS
          value: "1"
```

##### Horizontal Pod Autoscaler (HPA) Compatibility

**⚠️ WARNING: HPA is NOT compatible with partitioned projections.**

**Why:** Partitioning requires a fixed `TOTAL_PARTITIONS` value. Each worker must be configured with a specific `PARTITION_KEY` from 0 to `TOTAL_PARTITIONS-1`. When HPA scales pods up or down dynamically:

1. **Scaling up**: New pods don't automatically get assigned unique partition keys
2. **Scaling down**: Removed pods leave gaps in partition coverage
3. **Result**: Events may be skipped or processed multiple times

**Alternatives to HPA:**

1. **Pre-plan capacity**: Set `replicas` to expected max load
2. **Manual scaling**: Scale StatefulSet manually when needed:
   ```bash
   kubectl scale statefulset projection-workers --replicas=8
```
3. **Vertical scaling**: Use VPA (Vertical Pod Autoscaler) to adjust resource requests/limits
4. **Different projections, different scales**: Run fast projections without partitioning, slow projections with fixed partitions

**If you need dynamic scaling:**
- Run projections without partitioning (`TOTAL_PARTITIONS=1`)
- Scale the single projection vertically (more CPU/memory)
- Or split into multiple independent projections that can scale separately

#### Systemd

```ini
# /etc/systemd/system/projection-worker@.service
[Unit]
Description=Projection Worker %i
After=network.target postgresql.service
Wants=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment="DATABASE_URL=postgres://myapp:password@localhost/myapp"
Environment="PARTITION_KEY=%i"
Environment="TOTAL_PARTITIONS=4"
ExecStart=/opt/myapp/bin/myapp run-projections
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=projection-worker-%i

# Security hardening
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable projection-worker@{0..3}
sudo systemctl start projection-worker@{0..3}

# Check status
sudo systemctl status projection-worker@*

# View logs
sudo journalctl -u projection-worker@0 -f
```

### Pattern 2: Separate Services per Projection

Run different projections in separate deployments for better isolation.

```yaml
# docker-compose.yml
version: '3.8'

services:
  user-projection:
    image: myapp:latest
    environment:
      PROJECTION_NAME: "user_read_model"
      DATABASE_URL: "postgres://..."
    command: ["./myapp", "run-projection", "--name=user_read_model"]
    restart: unless-stopped

  analytics-projection:
    image: myapp:latest
    environment:
      PROJECTION_NAME: "analytics"
      DATABASE_URL: "postgres://..."
      WORKERS: "4"  # Scale this projection
    command: ["./myapp", "run-projection", "--name=analytics", "--workers=4"]
    restart: unless-stopped

  notification-projection:
    image: myapp:latest
    environment:
      PROJECTION_NAME: "notifications"
      DATABASE_URL: "postgres://..."
    command: ["./myapp", "run-projection", "--name=notifications"]
    restart: unless-stopped
```

### Pattern 3: Combined Deployment

Run multiple projections in the same process, separate processes for partitioned ones.

```go
func main() {
    if len(os.Args) > 1 && os.Args[1] == "projections" {
        runProjections()
    } else {
        runWebServer()
    }
}

func runProjections() {
    // Parse flags
    partitionKey := flag.Int("partition-key", -1, "Partition key for scaled projections")
    flag.Parse()
    
    store := postgres.NewStore(postgres.DefaultStoreConfig())
    
    var runners []runner.ProjectionRunner
    
    // Fast projections - run unpartitioned
    config1 := projection.DefaultProcessorConfig()
    processor1 := postgres.NewProcessor(db, store, &config1)
    runners = append(runners, runner.ProjectionRunner{
        Projection: &FastProjection1{},
        Processor:  processor1,
    })
    
    config2 := projection.DefaultProcessorConfig()
    processor2 := postgres.NewProcessor(db, store, &config2)
    runners = append(runners, runner.ProjectionRunner{
        Projection: &FastProjection2{},
        Processor:  processor2,
    })
    
    // Slow projection - only if partition key provided
    if *partitionKey >= 0 {
        config := projection.DefaultProcessorConfig()
        config.PartitionKey = *partitionKey
        config.TotalPartitions = 4
        
        processor := postgres.NewProcessor(db, store, &config)
        runners = append(runners, runner.ProjectionRunner{
            Projection: &SlowProjection{},
            Processor:  processor,
        })
    }
    
    r := runner.New()
    r.Run(ctx, runners)
}
```

## Monitoring

For comprehensive observability including logging, tracing, and metrics, see the [Observability Guide](./observability.md).

### Key Metrics to Track

1. **Projection Lag**
```sql
-- How far behind is each projection?
SELECT 
    projection_name,
    last_global_position,
    (SELECT MAX(global_position) FROM events) - last_global_position as lag,
    updated_at
FROM projection_checkpoints
ORDER BY lag DESC;
```

**Note:** For production monitoring, consider implementing dedicated observability projections that maintain pre-computed metrics tables, rather than running expensive analytical queries directly against the events table.

### Prometheus Metrics

Instrument your application:

```go
import "github.com/prometheus/client_golang/prometheus"

var (
    projectionLag = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "pupsourcing_projection_lag",
            Help: "Number of events projection is behind",
        },
        []string{"projection_name"},
    )
    
    eventsProcessed = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "pupsourcing_events_processed_total",
            Help: "Total number of events processed",
        },
        []string{"projection_name"},
    )
    
    projectionErrors = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "pupsourcing_projection_errors_total",
            Help: "Total number of projection errors",
        },
        []string{"projection_name"},
    )
)

func init() {
    prometheus.MustRegister(projectionLag)
    prometheus.MustRegister(eventsProcessed)
    prometheus.MustRegister(projectionErrors)
}

// Update metrics in your projection wrapper
type InstrumentedProjection struct {
    inner projection.Projection
}

func (p *InstrumentedProjection) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    err := p.inner.Handle(ctx, tx, event)
    if err != nil {
        projectionErrors.WithLabelValues(p.inner.Name()).Inc()
        return err
    }
    eventsProcessed.WithLabelValues(p.inner.Name()).Inc()
    return nil
}
```

### Grafana Dashboard

Example Prometheus queries:

```promql
# Projection lag
pupsourcing_projection_lag

# Event processing rate (events/sec)
rate(pupsourcing_events_processed_total[1m])

# Error rate
rate(pupsourcing_projection_errors_total[5m])

# Lag as percentage of total events
(pupsourcing_projection_lag / on() group_left() 
  max(pg_stat_user_tables_n_tup_ins{table="events"})) * 100
```

### Health Checks

```go
func healthCheck(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Check database connectivity
        if err := db.PingContext(r.Context()); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]string{
                "status": "unhealthy",
                "error": err.Error(),
            })
            return
        }
        
        // Check projection lag
        var maxLag int64
        err := db.QueryRowContext(r.Context(),
            `SELECT COALESCE(MAX((SELECT MAX(global_position) FROM events) - last_global_position), 0)
             FROM projection_checkpoints`).Scan(&maxLag)
        if err != nil || maxLag > 10000 {  // Threshold
            w.WriteHeader(http.StatusServiceUnavailable)
            json.NewEncoder(w).Encode(map[string]interface{}{
                "status": "unhealthy",
                "lag": maxLag,
                "threshold": 10000,
            })
            return
        }
        
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(map[string]string{
            "status": "healthy",
        })
    }
}
```

## Operational Best Practices

### Database Connection Pooling

```go
db, _ := sql.Open("postgres", connStr)

// Configure connection pool
db.SetMaxOpenConns(25)  // Max concurrent connections
db.SetMaxIdleConns(5)   // Idle connections to keep alive
db.SetConnMaxLifetime(5 * time.Minute)
db.SetConnMaxIdleTime(1 * time.Minute)
```

**Guidelines:**
- `MaxOpenConns` = number of workers × 2 (for safety)
- Monitor `pg_stat_activity` to verify connection usage
- Adjust based on PostgreSQL `max_connections`

### Graceful Shutdown

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    // Signal handling
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
    
    // Run projections in goroutine
    errChan := make(chan error, 1)
    go func() {
        errChan <- runner.RunProjections(ctx, db, store, configs)
    }()
    
    // Wait for signal or error
    select {
    case <-sigChan:
        log.Println("Shutdown signal received")
        cancel()
        
        // Wait for graceful shutdown with timeout
        shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer shutdownCancel()
        
        select {
        case <-errChan:
            log.Println("Projections stopped gracefully")
        case <-shutdownCtx.Done():
            log.Println("Shutdown timeout exceeded")
        }
        
    case err := <-errChan:
        log.Printf("Projection error: %v", err)
    }
}
```

### Backup and Recovery

**Backup Strategy:**
1. Regular PostgreSQL backups (pg_dump or continuous archiving)
2. Keep backups for retention period (e.g., 30 days)
3. Test recovery procedures regularly

**Recovery Scenarios:**

**Scenario 1: Lost Checkpoint**
```sql
-- Projection will restart from position 0
-- Ensure projections are idempotent!
DELETE FROM projection_checkpoints WHERE projection_name = 'lost_projection';
```

**Scenario 2: Corrupted Read Model**
```sql
-- 1. Stop projection
-- 2. Clear read model
TRUNCATE TABLE my_read_model;

-- 3. Delete checkpoint
DELETE FROM projection_checkpoints WHERE projection_name = 'my_projection';

-- 4. Restart projection (rebuilds from scratch)
```

### Security Considerations

1. **Database Credentials**
   - Use connection pooling
   - Rotate credentials regularly
   - Store in secrets management (Vault, AWS Secrets Manager)

2. **Network Security**
   - Use SSL/TLS for database connections (`sslmode=require`)
   - Firewall rules to restrict access
   - VPC isolation in cloud environments

3. **Least Privilege**

**Basic Example:**
```sql
-- Create read-only user for read models
CREATE USER cqrs_reader WITH PASSWORD '...';
GRANT USAGE ON SCHEMA public TO cqrs_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO cqrs_reader;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO cqrs_reader;

-- Set default privileges for future tables and sequences
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT SELECT ON TABLES TO cqrs_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT SELECT ON SEQUENCES TO cqrs_reader;

-- Create read/write user for projections
CREATE USER cqrs_writer WITH PASSWORD '...';
GRANT USAGE ON SCHEMA public TO cqrs_writer;
GRANT SELECT, INSERT, UPDATE ON events TO cqrs_writer;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO cqrs_writer;
GRANT ALL ON projection_checkpoints TO cqrs_writer;
GRANT ALL ON my_read_model TO cqrs_writer;

-- Set default privileges for future tables and sequences
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO cqrs_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA public 
    GRANT USAGE, SELECT ON SEQUENCES TO cqrs_writer;
```

**Advanced Example with Schema Separation (Recommended for Production):**

This approach enforces event sourcing hygiene by preventing updates/deletes on event streams while allowing projections full access to their data.

**⚠️ Important:** When using custom schemas, you must modify the SQL migration files generated by pupsourcing's `migrate-gen` tool. The generated migrations create tables in the `public` schema by default. You'll need to:
1. Generate the migrations: `go run github.com/getpup/pupsourcing/cmd/migrate-gen -output migrations`
2. Edit the generated SQL files to create/move tables to your custom schemas (`es`, `es_meta`, `projections`)
3. Apply the modified migrations using your migration tool

```sql
-- Create schemas for bounded contexts
CREATE SCHEMA es;        -- Event store (immutable events)
CREATE SCHEMA es_meta;   -- Event sourcing metadata (checkpoints, aggregate heads)
CREATE SCHEMA projections; -- Read models (mutable projections)

-- Move tables to appropriate schemas
ALTER TABLE events SET SCHEMA es;
ALTER TABLE aggregate_heads SET SCHEMA es_meta;
ALTER TABLE projection_checkpoints SET SCHEMA es_meta;
-- Move your read model tables to projections schema

-- Create read-only user (only sees projections)
CREATE USER cqrs_reader WITH PASSWORD '...';
GRANT USAGE ON SCHEMA projections TO cqrs_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA projections TO cqrs_reader;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA projections TO cqrs_reader;

ALTER DEFAULT PRIVILEGES IN SCHEMA projections 
    GRANT SELECT ON TABLES TO cqrs_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA projections 
    GRANT SELECT ON SEQUENCES TO cqrs_reader;

-- Create read/write user for event sourcing (write events, read/write projections)
CREATE USER cqrs_writer WITH PASSWORD '...';

-- Event store access (INSERT only - no updates/deletes for event sourcing hygiene)
GRANT USAGE ON SCHEMA es TO cqrs_writer;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA es TO cqrs_writer;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA es TO cqrs_writer;
REVOKE UPDATE, DELETE ON ALL TABLES IN SCHEMA es FROM cqrs_writer;

-- Metadata access (full access for checkpoints and aggregate heads)
GRANT USAGE ON SCHEMA es_meta TO cqrs_writer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA es_meta TO cqrs_writer;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA es_meta TO cqrs_writer;

-- Projections access (full access to build read models)
GRANT USAGE ON SCHEMA projections TO cqrs_writer;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA projections TO cqrs_writer;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA projections TO cqrs_writer;

-- Set default privileges for future tables and sequences
ALTER DEFAULT PRIVILEGES IN SCHEMA es 
    GRANT SELECT, INSERT ON TABLES TO cqrs_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA es 
    GRANT USAGE, SELECT ON SEQUENCES TO cqrs_writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA es_meta 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO cqrs_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA es_meta 
    GRANT USAGE, SELECT ON SEQUENCES TO cqrs_writer;

ALTER DEFAULT PRIVILEGES IN SCHEMA projections 
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO cqrs_writer;
ALTER DEFAULT PRIVILEGES IN SCHEMA projections 
    GRANT USAGE, SELECT ON SEQUENCES TO cqrs_writer;
```

**Why enforce INSERT-only on the event store?**
- Prevents accidental data corruption (events are immutable facts)
- Catches bugs early (attempting UPDATE/DELETE fails immediately)
- Maintains audit trail integrity (no event can ever be changed or removed)
- Forces proper event sourcing patterns (new events instead of modifying history)
- Database-level enforcement of event sourcing principles

## Troubleshooting

### Projection Falling Behind

**Symptoms:** Increasing lag, slow checkpoint updates

**Diagnosis:**
```sql
-- Check current lag
SELECT projection_name, 
       (SELECT MAX(global_position) FROM events) - last_global_position as lag
FROM projection_checkpoints;

-- Check processing rate
SELECT projection_name, updated_at
FROM projection_checkpoints
ORDER BY updated_at DESC;
```

**Solutions:**
1. **Increase batch size** - Process more events per transaction
2. **Add partitions** - Scale horizontally
3. **Optimize projection logic** - Profile and optimize slow code
4. **Check database performance** - Indexes, query plans

### High Database Load

**Symptoms:** Slow queries, connection pool exhaustion

**Diagnosis:**
```sql
-- Active connections
SELECT COUNT(*) FROM pg_stat_activity WHERE state = 'active';

-- Long-running queries
SELECT pid, now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;

-- Lock contention
SELECT * FROM pg_locks WHERE NOT granted;
```

**Solutions:**

1. Reduce `MaxOpenConns`
2. Decrease batch size
3. Add database indexes
4. Scale database (read replicas, sharding)
5. Increase poll interval to reduce database load:

```go
config := projection.DefaultProcessorConfig()
config.PollInterval = 500 * time.Millisecond  // Default is 100ms
// Higher values reduce database load but increase projection latency
// Trade-off: Less frequent polling = lower load, higher latency
```

### Stuck Projections

**Symptoms:** Checkpoint not updating, no errors

**Diagnosis:**
```bash
# Check if process is running
ps aux | grep myapp

# Check logs
journalctl -u projection-worker@0 -n 100

# Check for deadlocks
docker logs projection-worker-0 | grep -i deadlock
```

**Solutions:**
1. Restart projection process
2. Check for infinite loops in projection code
3. Verify database connectivity
4. Check for blocking locks in database

## See Also

- [Scaling Guide](./scaling.md) - Projection scaling patterns
- [Core Concepts](./core-concepts.md) - Understanding the architecture
- [Examples](https://github.com/getpup/pupsourcing/tree/master/examples) - Deployment examples
