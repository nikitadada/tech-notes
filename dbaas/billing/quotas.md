# Quotas

Ограничения на количество и объём ресурсов, доступных проекту/организации.

## Зачем квоты

- **Защита от случайного перерасхода** — предотвращение огромного счёта
- **Fair usage** — один tenant не занимает все ресурсы платформы
- **Capacity planning** — предсказуемое потребление ресурсов
- **Compliance** — ограничения по договору

## Типы квот

### Resource quotas

```json
{
  "project_id": "proj-456",
  "quotas": {
    "instances": {
      "max_count": 10,
      "current": 3
    },
    "cpu_total_vcpu": {
      "max": 32,
      "current": 6
    },
    "memory_total_gb": {
      "max": 64,
      "current": 12
    },
    "storage_total_gb": {
      "max": 1000,
      "current": 150
    },
    "backup_storage_gb": {
      "max": 500,
      "current": 45
    }
  }
}
```

### Rate quotas

```json
{
  "api_requests_per_minute": 600,
  "connections_per_instance": 500,
  "backup_operations_per_day": 10
}
```

## Проверка квот

```go
func (s *QuotaService) Check(ctx context.Context, projectID string, requested Resources) error {
    current, err := s.getCurrentUsage(ctx, projectID)
    if err != nil {
        return err
    }

    limits, err := s.getLimits(ctx, projectID)
    if err != nil {
        return err
    }

    if current.Instances+1 > limits.MaxInstances {
        return ErrQuotaExceeded("instances", limits.MaxInstances)
    }
    if current.CPU+requested.CPU > limits.MaxCPU {
        return ErrQuotaExceeded("cpu", limits.MaxCPU)
    }
    if current.Storage+requested.Storage > limits.MaxStorage {
        return ErrQuotaExceeded("storage", limits.MaxStorage)
    }

    return nil
}
```

### Race condition при проверке

Два параллельных запроса могут пройти проверку до обновления счётчика.

```go
// Решение: атомарная резервация
func (s *QuotaService) Reserve(ctx context.Context, projectID string, resources Resources) (ReleaseFunc, error) {
    // UPDATE quotas SET current_cpu = current_cpu + $1
    // WHERE project_id = $2 AND current_cpu + $1 <= max_cpu
    // RETURNING current_cpu

    rows, err := s.db.Exec(ctx, `
        UPDATE project_quotas
        SET current_cpu = current_cpu + $1,
            current_instances = current_instances + 1
        WHERE project_id = $2
          AND current_cpu + $1 <= max_cpu
          AND current_instances + 1 <= max_instances
    `, resources.CPU, projectID)

    if rows == 0 {
        return nil, ErrQuotaExceeded(...)
    }

    // Release function: вызывается при ошибке provisioning
    release := func() {
        s.db.Exec(ctx, `
            UPDATE project_quotas
            SET current_cpu = current_cpu - $1,
                current_instances = current_instances - 1
            WHERE project_id = $2
        `, resources.CPU, projectID)
    }

    return release, nil
}
```

## API для управления квотами

```
# Получить текущие квоты
GET /v1/projects/{id}/quotas

# Запросить увеличение (для пользователя)
POST /v1/projects/{id}/quota-requests
{
  "resource": "instances",
  "requested_limit": 20,
  "justification": "scaling for new product launch"
}

# Установить квоту (для admin)
PUT /v1/admin/projects/{id}/quotas
{
  "instances": {"max": 20},
  "cpu_total_vcpu": {"max": 64}
}
```

## Default Quotas

| Tier | Instances | CPU | Memory | Storage |
|------|-----------|-----|--------|---------|
| Free | 1 | 1 vCPU | 1 GB | 5 GB |
| Standard | 10 | 32 vCPU | 64 GB | 1 TB |
| Business | 50 | 128 vCPU | 256 GB | 10 TB |
| Enterprise | Custom | Custom | Custom | Custom |

## Уведомления о квотах

```
80% usage → Warning notification
90% usage → Critical notification
100% usage → Block new resources, notify

Email: "Your project proj-456 is using 85% of CPU quota (27/32 vCPU)"
```
