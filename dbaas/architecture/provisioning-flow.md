# Provisioning Flow

Процесс создания нового инстанса БД от API-запроса до готовности.

## Общий flow

```
User Request
    │
    ▼
┌─────────────┐
│  API Layer  │ Validate, check quotas, create operation
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Scheduler  │ Choose region, zone, host
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Provisioner │ Allocate compute, storage, network
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Deployer   │ Install DB engine, configure, start
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Setup     │ Create users, extensions, replication
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Register   │ DNS, endpoints, monitoring, backup schedule
└──────┬──────┘
       │
       ▼
    Instance RUNNING
```

## Детализация шагов

### 1. API Validation

```go
func (s *Service) CreateInstance(ctx context.Context, req *CreateInstanceRequest) (*Operation, error) {
    // Validate request
    if err := req.Validate(); err != nil {
        return nil, status.Errorf(codes.InvalidArgument, "invalid request: %v", err)
    }

    // Check quotas
    if err := s.quotaService.Check(ctx, req.ProjectID, req.Resources); err != nil {
        return nil, status.Errorf(codes.ResourceExhausted, "quota exceeded: %v", err)
    }

    // Check idempotency key
    if op, err := s.opStore.GetByIdempotencyKey(ctx, req.IdempotencyKey); err == nil {
        return op, nil // already created
    }

    // Create instance record
    instance := &Instance{
        ID:        generateID(),
        ProjectID: req.ProjectID,
        Name:      req.Name,
        Engine:    req.Engine,
        Version:   req.Version,
        Resources: req.Resources,
        State:     StatePending,
    }

    // Create async operation
    op := s.createOperation(ctx, instance, OperationCreate)
    return op, nil
}
```

### 2. Scheduling

```go
type ScheduleRequest struct {
    Region    string
    Resources Resources
    HA        bool           // нужен multi-AZ
    Affinity  []Constraint   // constraints
}

type ScheduleResult struct {
    PrimaryPlacement Placement
    ReplicaPlacements []Placement // для HA
}

func (s *Scheduler) Schedule(req ScheduleRequest) (*ScheduleResult, error) {
    // 1. Фильтрация: зоны с достаточными ресурсами
    candidates := s.filterByResources(req.Region, req.Resources)

    // 2. Anti-affinity: primary и replicas в разных зонах
    if req.HA {
        candidates = s.applyAntiAffinity(candidates)
    }

    // 3. Скоринг: bin-packing, cost, load balancing
    ranked := s.scoreAndRank(candidates)

    return &ScheduleResult{
        PrimaryPlacement:  ranked[0],
        ReplicaPlacements: ranked[1:],
    }, nil
}
```

### 3. Resource Allocation

В зависимости от инфраструктуры:

**Kubernetes:**
```yaml
# Создаётся StatefulSet + PVC + Service + ConfigMap
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: inst-abc123
  labels:
    instance-id: abc123
    project-id: proj-456
spec:
  replicas: 1
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 50Gi
```

**VM-based:**
```
1. Create VM (API call to cloud provider)
2. Attach disk
3. Configure network (VPC, security groups)
4. Wait for VM ready
5. SSH / cloud-init setup
```

### 4. DB Engine Setup

```bash
# Инициализация (через Agent или init container)
initdb --data-checksums -D /var/lib/postgresql/data

# Применение конфигурации
postgresql.conf:
  shared_buffers = '1GB'
  max_connections = 200
  wal_level = replica
  max_wal_senders = 10

# Создание пользователей
CREATE ROLE app_user WITH LOGIN PASSWORD '***';
GRANT ALL ON DATABASE userdb TO app_user;

# Настройка extensions
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

### 5. Replication Setup (для HA)

```
Primary (zone-a) → Streaming Replication → Replica (zone-b)
                                        → Replica (zone-c)
```

### 6. Registration

- DNS запись: `inst-abc123.db.example.com → Load Balancer IP`
- Endpoint в metadata DB: host, port, role
- Backup schedule: cron-based или continuous WAL archiving
- Monitoring: register targets в Prometheus/Victoria

## Время provisioning

| Этап | VM-based | Kubernetes |
|------|----------|------------|
| VM/Pod creation | 1-3 min | 10-30 sec |
| Disk allocation | 30-60 sec | 5-10 sec |
| DB init | 5-15 sec | 5-15 sec |
| Replication setup | 30-60 sec | 30-60 sec |
| **Итого** | **2-5 min** | **1-2 min** |

## Error Handling

Каждый шаг должен быть:
- **Idempotent** — повторный вызов безопасен
- **Reversible** — при ошибке на шаге N откатываем шаги N-1...1
- **Retriable** — transient ошибки ретраятся автоматически

```go
type Step struct {
    Name    string
    Execute func(ctx context.Context) error
    Rollback func(ctx context.Context) error
}

// Saga pattern для provisioning
steps := []Step{
    {Name: "allocate_compute", Execute: allocateVM, Rollback: deallocateVM},
    {Name: "allocate_storage", Execute: createDisk, Rollback: deleteDisk},
    {Name: "setup_network", Execute: configureNet, Rollback: cleanupNet},
    {Name: "deploy_db", Execute: deployDB, Rollback: removeDB},
}
```
