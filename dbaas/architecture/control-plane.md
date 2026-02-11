# Control Plane

Управляющая плоскость DBaaS. Центральный компонент, координирующий все операции над инстансами.

## Компоненты

### API Service

Входная точка для всех управляющих запросов.

```
Client → API Gateway → Auth → Rate Limiter → API Service → Orchestrator
```

- REST или gRPC API
- Валидация запросов (размеры, конфигурация, квоты)
- Создание async operation (task) для длительных действий
- Возвращает operation ID для отслеживания

### Orchestrator (Instance Manager)

Центральный координатор. Управляет state machine каждого инстанса.

```go
type InstanceState string

const (
    StatePending   InstanceState = "PENDING"
    StateCreating  InstanceState = "CREATING"
    StateRunning   InstanceState = "RUNNING"
    StateModifying InstanceState = "MODIFYING"
    StateStopping  InstanceState = "STOPPING"
    StateStopped   InstanceState = "STOPPED"
    StateDeleting  InstanceState = "DELETING"
    StateDeleted   InstanceState = "DELETED"
    StateError     InstanceState = "ERROR"
)
```

Каждый переход — набор шагов (workflow):

```
CreateInstance workflow:
  1. Validate params
  2. Reserve quota
  3. Allocate resources (compute, storage, network)
  4. Deploy DB engine
  5. Configure replication
  6. Setup monitoring
  7. Register DNS
  8. Mark as RUNNING
```

### Scheduler

Выбирает placement для нового инстанса.

```go
type PlacementDecision struct {
    Region string
    Zone   string
    Host   string
    Score  float64
}

// Факторы скоринга
// - Available resources (CPU, RAM, disk)
// - Anti-affinity constraints
// - Data locality
// - Cost optimization
// - Failure domain diversity
```

### Config Manager

Управляет конфигурацией СУБД.

- Хранит default + user overrides
- Применяет конфиг через Agent
- Различает параметры, требующие рестарта и dynamic
- Валидирует значения (min/max, совместимость)

### Health Monitor

Агрегирует health status от всех инстансов.

```
Agent → heartbeat (каждые 10-30s) → Health Monitor
  - DB process status
  - Replication lag
  - Disk usage
  - CPU/Memory usage

Health Monitor → triggers:
  - Failover при недоступности primary
  - Alert при degraded state
  - Auto-scaling при high usage
```

## Metadata Database

CP хранит всё состояние в собственной БД (обычно PostgreSQL или etcd).

```sql
-- Core tables
instances       -- id, project_id, name, state, config, created_at
operations      -- id, instance_id, type, state, steps, started_at
backups         -- id, instance_id, type, size, state, created_at
endpoints       -- id, instance_id, host, port, role (primary/replica)
maintenance     -- id, instance_id, window, pending_operations
```

**Критично:** metadata DB — single point of failure для CP. Она сама должна быть HA (multi-AZ, replicated).

## Idempotency и Reconciliation

### Idempotent operations
Каждая операция идемпотентна. Повторный вызов безопасен — проверяет текущее состояние и пропускает уже выполненные шаги.

### Reconciliation loop
Периодически сверяет desired state (в metadata DB) с actual state (от Agent). При расхождении — корректирует.

```
Every N seconds:
  for each instance:
    desired = getDesiredState(instance)
    actual  = getActualState(instance)  // from agent
    if desired != actual:
      reconcile(instance, desired, actual)
```

## Масштабирование Control Plane

- Stateless API-сервисы → горизонтальное масштабирование
- Шардирование orchestrator'а по tenant/region
- Event-driven архитектура (Kafka/NATS) для loose coupling
- Metadata DB — вертикальное масштабирование или шардирование
