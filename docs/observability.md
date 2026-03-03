# Observability

Logging, tracing, and monitoring capabilities for pupsourcing applications.

## Table of Contents

- [Overview](#overview)
- [Logging](#logging)
- [Distributed Tracing](#distributed-tracing)
- [Metrics](#metrics)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)
- [Related Documentation](#related-documentation)

## Overview

pupsourcing provides comprehensive observability features:

1. **Logging** - Optional logger injection without forced dependencies
2. **Distributed Tracing** - Built-in TraceID, CorrelationID, and CausationID support
3. **Metrics** - Integration patterns with monitoring systems

## Logging

### Logger Interface

Minimal interface enabling integration with any logging library:

```go
type Logger interface {
    Debug(ctx context.Context, msg string, keyvals ...interface{})
    Info(ctx context.Context, msg string, keyvals ...interface{})
    Error(ctx context.Context, msg string, keyvals ...interface{})
}
```

### Event Store Logging

Logs append operations, read operations, and concurrency conflicts:

```go
import "github.com/pupsourcing/core/es/adapters/postgres"

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

// Inject logger
config := postgres.DefaultStoreConfig()
config.Logger = &MyLogger{logger: slog.Default()}
store := postgres.NewStore(config)
```

### Consumer Logging

Logs worker and processor lifecycle, batch progress, checkpoints, and errors:

```go
import (
    "github.com/pupsourcing/core/es/adapters/postgres"
    "github.com/pupsourcing/core/es/worker"
)

store := postgres.NewStore(postgres.DefaultStoreConfig())
w := postgres.NewWorker(
    db,
    store,
    worker.WithLogger(&MyLogger{logger: slog.Default()}),
)
```

### Zero-Overhead Design

Logging disabled by default with no performance impact:

```go
// No logger configured = zero overhead
config := postgres.DefaultStoreConfig()  // Logger is nil
store := postgres.NewStore(config)
```

All logging operations check `logger != nil` before execution, ensuring zero allocation or call overhead when disabled.

### Integration Examples

#### Standard Library log

```go
import "log"

type StdLogger struct{}

func (l *StdLogger) Debug(ctx context.Context, msg string, keyvals ...interface{}) {
    log.Printf("[DEBUG] %s %v", msg, keyvals)
}

func (l *StdLogger) Info(ctx context.Context, msg string, keyvals ...interface{}) {
    log.Printf("[INFO] %s %v", msg, keyvals)
}

func (l *StdLogger) Error(ctx context.Context, msg string, keyvals ...interface{}) {
    log.Printf("[ERROR] %s %v", msg, keyvals)
}
```

#### slog (Go 1.21+)

```go
import "log/slog"

type SlogLogger struct {
    logger *slog.Logger
}

func (l *SlogLogger) Debug(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.DebugContext(ctx, msg, keyvals...)
}

func (l *SlogLogger) Info(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.InfoContext(ctx, msg, keyvals...)
}

func (l *SlogLogger) Error(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.ErrorContext(ctx, msg, keyvals...)
}
```

#### zap

```go
import "go.uber.org/zap"

type ZapLogger struct {
    logger *zap.SugaredLogger
}

func (l *ZapLogger) Debug(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.Debugw(msg, keyvals...)
}

func (l *ZapLogger) Info(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.Infow(msg, keyvals...)
}

func (l *ZapLogger) Error(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.Errorw(msg, keyvals...)
}
```

#### zerolog

```go
import "github.com/rs/zerolog"

type ZerologLogger struct {
    logger zerolog.Logger
}

func (l *ZerologLogger) Debug(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.Debug().Fields(keyvals).Msg(msg)
}

func (l *ZerologLogger) Info(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.Info().Fields(keyvals).Msg(msg)
}

func (l *ZerologLogger) Error(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.Error().Fields(keyvals).Msg(msg)
}
```

#### logrus

```go
import "github.com/sirupsen/logrus"

type LogrusLogger struct {
    logger *logrus.Logger
}

func (l *LogrusLogger) Debug(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.WithFields(toLogrusFields(keyvals)).Debug(msg)
}

func (l *LogrusLogger) Info(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.WithFields(toLogrusFields(keyvals)).Info(msg)
}

func (l *LogrusLogger) Error(ctx context.Context, msg string, keyvals ...interface{}) {
    l.logger.WithFields(toLogrusFields(keyvals)).Error(msg)
}

func toLogrusFields(keyvals []interface{}) logrus.Fields {
    fields := make(logrus.Fields)
    for i := 0; i < len(keyvals); i += 2 {
        if i+1 < len(keyvals) {
            fields[fmt.Sprint(keyvals[i])] = keyvals[i+1]
        }
    }
    return fields
}
```

See the [with-logging example](https://github.com/pupsourcing/core/tree/master/examples/with-logging) for a complete working demonstration.

## Distributed Tracing

pupsourcing includes built-in support for distributed tracing through three optional string fields in every event:

- **TraceID** - Links all events in a distributed operation (e.g., a user request across multiple services)
- **CorrelationID** - Links related events across aggregates within the same business transaction
- **CausationID** - Identifies the event or command that caused this event

These fields accept any string format (UUID, ULID, or custom IDs) for maximum flexibility.

### Using Trace IDs

Extract the trace ID from your request context and propagate it to events:

```go
import (
    "go.opentelemetry.io/otel/trace"
    "github.com/google/uuid"
)

func HandleRequest(ctx context.Context, store *postgres.Store) error {
    // Extract OpenTelemetry trace ID from context
    span := trace.SpanFromContext(ctx)
    traceID := span.SpanContext().TraceID()
    
    // Convert to string - OpenTelemetry trace IDs are 128-bit hex strings
    traceIDStr := traceID.String()
    
    // Create event with trace ID
    event := es.Event{
        AggregateType: "Order",
        AggregateID:   orderID,
        EventID:       uuid.New(),
        EventType:     "OrderCreated",
        EventVersion:  1,
        Payload:       payload,
        Metadata:      []byte(`{}`),
        CreatedAt:     time.Now(),
        TraceID:       es.NullString{String: traceIDStr, Valid: true},
    }
    
    tx, _ := db.BeginTx(ctx, nil)
    defer tx.Rollback()
    
    _, err := store.Append(ctx, tx, es.Any(), []es.Event{event})
    if err != nil {
        return err
    }
    
    return tx.Commit()
}
```

### Propagating Trace Context in Consumers

When processing events in consumers, propagate the trace ID to maintain observability:

```go
type TracedConsumer struct {
    tracer trace.Tracer
}

func (p *TracedConsumer) Handle(ctx context.Context, tx *sql.Tx, event es.PersistedEvent) error {
    // Extract trace ID from event if present
    if event.TraceID.Valid {
        traceID, err := trace.TraceIDFromHex(event.TraceID.String)
        if err == nil {
            spanCtx := trace.NewSpanContext(trace.SpanContextConfig{
                TraceID:    traceID,
                TraceFlags: trace.FlagsSampled,
            })
            ctx = trace.ContextWithSpanContext(ctx, spanCtx)
        }
    }
    
    // Start a new span for consumer processing
    ctx, span := p.tracer.Start(ctx, "consumer.handle",
        trace.WithAttributes(
            attribute.String("event.type", event.EventType),
            attribute.String("aggregate.type", event.AggregateType),
            attribute.String("aggregate.id", event.AggregateID),
        ),
    )
    defer span.End()
    
    // Process event with trace context using processor's transaction
    _, err := tx.ExecContext(ctx, "INSERT INTO traced_events ...")
    return err
}
```

### Correlation and Causation

Use CorrelationID and CausationID to track event relationships:

```go
// Generate a correlation ID for the business transaction
correlationID := uuid.New().String()

// Original command creates first event
originalEvent := es.Event{
    EventID:       uuid.New(),
    AggregateID:   orderID,
    EventType:     "OrderCreated",
    CorrelationID: es.NullString{String: correlationID, Valid: true},
    // ... other fields
}

// Subsequent event caused by the first
followUpEvent := es.Event{
    EventID:       uuid.New(),
    AggregateID:   inventoryID,
    EventType:     "InventoryReserved",
    CorrelationID: es.NullString{String: correlationID, Valid: true},
    CausationID:   es.NullString{String: originalEvent.EventID.String(), Valid: true},
    // ... other fields
}
```

This creates a clear chain of causality:

- `CorrelationID` links all events in the same business transaction
- `CausationID` shows which event triggered this one

## Metrics

For metrics integration with Prometheus and other monitoring systems, see the [Deployment Guide](./deployment.md).

Key metrics to track:

- Event append rate
- Event append latency
- Consumer lag (events behind)
- Consumer processing rate
- Consumer errors

## Best Practices

### Logging

1. **Use appropriate log levels**
   - Debug: Detailed diagnostic information
   - Info: Significant operational events
   - Error: Error conditions that require attention

2. **Include context**
   - Always pass the context to logging methods
   - Include relevant key-value pairs (aggregate IDs, event types, etc.)

3. **Avoid PII in logs**
   - Don't log sensitive user data
   - Consider redacting event payloads

### Tracing

1. **Always propagate trace context**
   - Extract trace IDs from incoming requests
   - Include trace IDs in all events
   - Propagate to downstream services

2. **Use correlation IDs for business transactions**
   - Generate at the start of a business transaction
   - Include in all related events across aggregates

3. **Track causation chains**
   - Set CausationID when one event triggers another
   - Helps debug complex event chains

### Metrics

1. **Monitor consumer lag**
   - Alert when consumers fall too far behind
   - Critical for user-facing read models

2. **Track error rates**
   - Monitor consumer failures
   - Alert on sustained error conditions

3. **Measure latencies**
   - P50, P95, P99 for event appends
   - Consumer processing time per event

## Troubleshooting

### High Consumer Lag

Check:
1. Consumer processing performance (slow queries?)
2. Batch size configuration
3. Need for more workers (horizontal scaling)
4. Database connection pool size

### Optimistic Concurrency Conflicts

The logger will show these as ERROR level with aggregate details. Common causes:
1. Multiple services writing to same aggregate
2. Retry logic without backoff
3. Race conditions in application code

### Missing Events in Consumers

Use logging to verify:
1. Events are being appended (check store logs)
2. Consumer is processing (check worker/processor logs)
3. Segment ownership is healthy (`consumer_segments` ownership)

## Related Documentation

- [Deployment Guide](./deployment.md) - Production deployment and monitoring
- [Examples](https://github.com/pupsourcing/core/tree/master/examples/with-logging) - Complete logging example
- [Consumers](./consumers.md) - Consumer processing patterns
