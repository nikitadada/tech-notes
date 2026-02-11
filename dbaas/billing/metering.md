# Metering

Учёт потребления ресурсов для биллинга. Metering = измерение, Billing = начисление.

## Что измеряем

| Ресурс | Единица | Как считаем |
|--------|---------|-------------|
| Compute (CPU) | vCPU-hours | Время × кол-во vCPU |
| Compute (RAM) | GB-hours | Время × объём RAM |
| Storage | GB-months | Среднее использование за период |
| Backup storage | GB-months | Объём хранимых бэкапов |
| Network egress | GB | Объём исходящего трафика |
| IOPS | operations | Кол-во I/O операций (для provisioned IOPS) |
| Connections | count | Количество активных соединений |

## Архитектура metering

```
Instance (Data Plane)
  │
  Agent → usage events → Message Queue (Kafka)
                              │
                         ┌────┴────┐
                         │ Metering│
                         │ Service │
                         └────┬────┘
                              │
                    ┌─────────┼─────────┐
                    │         │         │
              ┌─────┴──┐ ┌───┴────┐ ┌──┴──────┐
              │ Usage  │ │ Rating │ │ Billing │
              │  Store │ │ Engine │ │ Service │
              └────────┘ └────────┘ └─────────┘
```

### Usage Event

```json
{
  "instance_id": "inst-abc123",
  "project_id": "proj-456",
  "timestamp": "2026-02-11T10:00:00Z",
  "period": 3600,
  "resources": {
    "cpu_vcpu": 2,
    "memory_gb": 4,
    "storage_gb": 52.3,
    "backup_storage_gb": 15.7,
    "network_egress_bytes": 1073741824,
    "connections_avg": 42
  }
}
```

### Сбор данных

```go
// Agent: периодический сбор usage
func (a *Agent) collectUsage(ctx context.Context) (*UsageEvent, error) {
    event := &UsageEvent{
        InstanceID: a.instanceID,
        Timestamp:  time.Now(),
        Period:     time.Hour,
    }

    // CPU/Memory: из cgroups или Kubernetes metrics
    event.CPU = getCPUAllocation()
    event.Memory = getMemoryAllocation()

    // Storage: du или df
    event.Storage = getDiskUsage("/var/lib/postgresql/data")

    // Network: from /proc/net/dev or Kubernetes metrics
    event.NetworkEgress = getNetworkEgress()

    // Connections: from pg_stat_activity
    event.Connections = getActiveConnections()

    return event, nil
}
```

## Гранулярность

| Подход | Описание | Точность | Сложность |
|--------|----------|----------|-----------|
| Per-hour | Snapshot каждый час | Средняя | Низкая |
| Per-minute | Snapshot каждую минуту | Высокая | Средняя |
| Event-based | Событие при каждом изменении | Максимальная | Высокая |

Большинство DBaaS используют per-hour metering. Serverless — per-second или per-request.

## Idempotency

Metering events могут дублироваться (retry, at-least-once delivery). Нужна дедупликация.

```go
type UsageEvent struct {
    ID         string    `json:"id"`          // unique event ID
    InstanceID string    `json:"instance_id"`
    Timestamp  time.Time `json:"timestamp"`
    // ... resources
}

// Deduplication: INSERT ON CONFLICT DO NOTHING
// или idempotency key в message queue
```

## Rating (тарификация)

Применение тарифа к usage данным.

```go
type Rate struct {
    Resource    string  // "cpu_vcpu_hour"
    PricePerUnit float64 // 0.05 USD
    Currency    string  // "USD"
}

func calculateCost(usage *UsageEvent, rates []Rate) float64 {
    var total float64
    total += usage.CPU * hours * rates["cpu_vcpu_hour"].PricePerUnit
    total += usage.Memory * hours * rates["memory_gb_hour"].PricePerUnit
    total += usage.Storage * rates["storage_gb_month"].PricePerUnit / 720
    total += usage.NetworkEgress / 1e9 * rates["network_egress_gb"].PricePerUnit
    return total
}
```

## Аккуратность учёта

- **Clock skew** — синхронизация времени через NTP
- **Missing data** — interpolation или пессимистичная оценка
- **Instance lifecycle** — считать с момента RUNNING, прекращать при STOPPED/DELETED
- **Stopped instances** — compute не тарифицируется, storage тарифицируется
