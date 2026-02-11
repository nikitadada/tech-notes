# Create Instance

Создание нового инстанса БД — основная операция DBaaS.

## API Request

```json
POST /v1/projects/{project_id}/instances
{
  "name": "my-postgres",
  "engine": "postgresql",
  "version": "16",
  "region": "eu-west-1",
  "resources": {
    "cpu": 2,
    "memory_gb": 4,
    "storage_gb": 50,
    "storage_class": "ssd"
  },
  "ha": {
    "enabled": true,
    "replicas": 2
  },
  "network": {
    "public_access": false,
    "allowed_cidrs": ["10.0.0.0/8"]
  },
  "backup": {
    "enabled": true,
    "schedule": "0 3 * * *",
    "retention_days": 7
  },
  "parameters": {
    "max_connections": 200,
    "shared_buffers": "1GB"
  }
}
```

## API Response

```json
{
  "instance": {
    "id": "inst-abc123",
    "name": "my-postgres",
    "state": "CREATING",
    "engine": "postgresql",
    "version": "16",
    "endpoints": [],
    "created_at": "2026-02-11T10:00:00Z"
  },
  "operation": {
    "id": "op-xyz789",
    "type": "CREATE_INSTANCE",
    "state": "RUNNING",
    "progress": 0,
    "started_at": "2026-02-11T10:00:00Z"
  }
}
```

Операция асинхронная — клиент поллит status через `GET /v1/operations/{id}`.

## Workflow шаги

```
1. Validate        → проверка параметров, квот, permissions
2. Reserve quota   → резервирование ресурсов в биллинге
3. Schedule        → выбор placement (zone, host)
4. Allocate        → создание compute, storage, network
5. Deploy          → установка СУБД, применение конфигурации
6. Initialize      → initdb, создание пользователей, extensions
7. Setup HA        → настройка репликации (если ha.enabled)
8. Register        → DNS, endpoints, monitoring, backup schedule
9. Health check    → проверка доступности, replication lag
10. Complete       → state = RUNNING, notify user
```

## Валидация

```go
func (r *CreateInstanceRequest) Validate() error {
    var errs []error

    if r.Name == "" || len(r.Name) > 63 {
        errs = append(errs, errors.New("name must be 1-63 characters"))
    }
    if !validEngine(r.Engine) {
        errs = append(errs, fmt.Errorf("unsupported engine: %s", r.Engine))
    }
    if !validVersion(r.Engine, r.Version) {
        errs = append(errs, fmt.Errorf("unsupported version: %s %s", r.Engine, r.Version))
    }
    if r.Resources.CPU < 1 || r.Resources.CPU > 64 {
        errs = append(errs, errors.New("cpu must be 1-64"))
    }
    if r.Resources.StorageGB < 10 || r.Resources.StorageGB > 10000 {
        errs = append(errs, errors.New("storage must be 10-10000 GB"))
    }

    return errors.Join(errs...)
}
```

## Instance States

```
                    ┌──────────┐
          ┌────────│ CREATING │
          │        └────┬─────┘
          │             │ success
          │             ▼
          │        ┌──────────┐     modify      ┌───────────┐
    error │        │ RUNNING  │ ───────────────▶ │ MODIFYING │
          │        └────┬─────┘                  └─────┬─────┘
          │             │ delete                        │
          ▼             ▼                               │ success
    ┌──────────┐  ┌──────────┐                         │
    │  ERROR   │  │ DELETING │                         │
    └──────────┘  └────┬─────┘                         ▼
                       │                          ┌──────────┐
                       ▼                          │ RUNNING  │
                  ┌──────────┐                    └──────────┘
                  │ DELETED  │
                  └──────────┘
```

## Endpoints после создания

```json
{
  "endpoints": [
    {
      "role": "primary",
      "host": "inst-abc123-primary.db.example.com",
      "port": 5432,
      "database": "defaultdb",
      "ssl_required": true
    },
    {
      "role": "replica",
      "host": "inst-abc123-replica.db.example.com",
      "port": 5432,
      "database": "defaultdb",
      "ssl_required": true
    }
  ]
}
```

## Connection string

```
postgresql://app_user:***@inst-abc123-primary.db.example.com:5432/defaultdb?sslmode=require
```
