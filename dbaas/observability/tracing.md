# Tracing

Отслеживание пути запроса через компоненты системы. Полезно для диагностики latency и ошибок.

## Два контекста tracing в DBaaS

### 1. Platform Tracing (Control Plane)

Отслеживание управляющих операций: создание инстанса, бэкап, failover.

```
CreateInstance request
  │
  ├── [span] API validation (2ms)
  ├── [span] Quota check (5ms)
  ├── [span] Schedule placement (15ms)
  ├── [span] Allocate resources (45s)
  │     ├── [span] Create PVC (3s)
  │     ├── [span] Create StatefulSet (2s)
  │     └── [span] Wait for Pod ready (40s)
  ├── [span] Initialize database (8s)
  │     ├── [span] initdb (3s)
  │     ├── [span] Apply config (1s)
  │     └── [span] Create users (500ms)
  ├── [span] Setup replication (30s)
  └── [span] Register endpoints (2s)

Total: 83.5s
```

### 2. Query Tracing (Data Plane)

Отслеживание пути SQL-запроса через proxy → DB.

```
App request (SELECT * FROM users WHERE id = 42)
  │
  ├── [span] Connection pool acquire (1ms)
  ├── [span] Proxy routing (0.5ms)
  ├── [span] PostgreSQL query execution (3ms)
  │     ├── [span] Parse (0.1ms)
  │     ├── [span] Plan (0.5ms)
  │     └── [span] Execute (2.4ms)
  └── [span] Result transmission (0.3ms)

Total: 4.8ms
```

## Instrumentation

### OpenTelemetry (Go)

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

var tracer = otel.Tracer("dbaas-control-plane")

func (s *Service) CreateInstance(ctx context.Context, req *CreateReq) error {
    ctx, span := tracer.Start(ctx, "CreateInstance",
        trace.WithAttributes(
            attribute.String("instance.engine", req.Engine),
            attribute.String("instance.region", req.Region),
        ),
    )
    defer span.End()

    // Каждый шаг — child span
    if err := s.validate(ctx, req); err != nil {
        span.RecordError(err)
        span.SetStatus(codes.Error, err.Error())
        return err
    }

    if err := s.allocateResources(ctx, req); err != nil {
        span.RecordError(err)
        return err
    }

    // ...
    return nil
}
```

### Propagation через Agent

```
Control Plane → gRPC call (с trace context в metadata) → Agent
  Agent создаёт child spans для локальных операций
```

```go
// Agent: извлечение trace context из gRPC metadata
func (a *Agent) ExecuteBackup(ctx context.Context, req *BackupReq) error {
    ctx, span := tracer.Start(ctx, "agent.ExecuteBackup")
    defer span.End()

    span.AddEvent("starting pg_basebackup")
    if err := a.runPgBasebackup(ctx); err != nil {
        span.RecordError(err)
        return err
    }
    span.AddEvent("backup completed", trace.WithAttributes(
        attribute.Int64("backup.size_bytes", size),
    ))
    return nil
}
```

## Backend

| Система | Описание |
|---------|----------|
| Jaeger | Open-source, distributed tracing |
| Tempo (Grafana) | Cost-effective, S3-backed |
| Zipkin | Lightweight tracing |
| Datadog APM | Commercial, full observability |

## Корреляция с метриками и логами

```
Trace ID: abc-123-def-456
  │
  ├── Spans → визуализация в Jaeger/Tempo
  ├── Logs  → фильтр по trace_id в Loki: {trace_id="abc-123-def-456"}
  └── Metrics → exemplars в Prometheus → link to trace

Grafana: metrics dashboard → click exemplar → open trace → see logs
```

Единый `trace_id` пронизывает все три сигнала — метрики, логи, трейсы.

## Что трейсить в DBaaS

| Операция | Приоритет | Зачем |
|----------|-----------|-------|
| API requests | High | Latency, errors |
| Provisioning workflow | High | Долгие операции, debugging |
| Backup/restore | Medium | Timing, bottlenecks |
| Failover | High | Время переключения |
| Agent commands | Medium | Debugging agent issues |
| Health checks | Low | Только при проблемах |
