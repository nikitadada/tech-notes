# SLA, SLO, SLI

Фреймворк для определения и измерения надёжности сервиса.

## Определения

| Термин | Что это | Пример |
|--------|---------|--------|
| **SLI** (Service Level Indicator) | Метрика, которую мы измеряем | 99.95% запросов успешны |
| **SLO** (Service Level Objective) | Целевое значение SLI | SLI availability > 99.95% |
| **SLA** (Service Level Agreement) | Юридическое обязательство + последствия | 99.95% uptime, иначе credit |

```
SLI (что измеряем) → SLO (какой цели хотим) → SLA (что обещаем клиенту)
```

## SLI для DBaaS

### Availability

```
availability = (total_time - downtime) / total_time

Что считается downtime:
  - Невозможно выполнить write-запрос к primary
  - Невозможно установить новое соединение
  - Время ответа > threshold (например, 30 сек)

Что НЕ считается downtime:
  - Planned maintenance (в maintenance window)
  - Проблемы на стороне пользователя
  - Недоступность read replica (если primary работает)
```

### Latency

```
query_latency_p99 < 100ms

Измерение:
  - p50, p95, p99 query latency
  - Connection establishment time
  - Failover time
```

### Durability

```
durability = 1 - (data_loss_events / total_transactions)

Цель: 99.999999999% (11 nines) — ни один committed transaction не потерян
```

### Throughput

```
Поддержка заявленного QPS без деградации latency.
```

## SLO примеры для DBaaS

```yaml
slos:
  availability:
    target: 99.95%           # multi-AZ instance
    window: 30 days (rolling)
    exclusions:
      - planned_maintenance
      - customer_initiated_actions

  latency:
    write_p99: 50ms          # на уровне proxy
    read_p99: 20ms
    connection_time_p99: 1s

  durability:
    target: 99.999999999%    # 11 nines
    committed_transactions: "zero loss with sync replication"

  recovery:
    failover_time: < 30s     # automatic failover
    backup_rpo: < 5min       # with WAL archiving
    backup_rto: < 1h         # full restore
```

## Error Budget

```
Error budget = 1 - SLO

SLO = 99.95% → Error budget = 0.05%
За 30 дней: 0.05% × 30 × 24 × 60 = 21.6 минут допустимого downtime

Если бюджет исчерпан:
  - Стоп на feature work
  - Фокус на reliability
  - Заморозка рискованных изменений
```

```
Budget consumed:
  ████████████████░░░░ 80% (17.3 min used / 21.6 min total)
  ↑ Warning threshold
```

## SLA в коммерческих DBaaS

| Provider | Service | SLA | Компенсация |
|----------|---------|-----|-------------|
| AWS RDS | Multi-AZ | 99.95% | 10-100% credit |
| Google Cloud SQL | HA | 99.95% | 10-50% credit |
| Azure Database | Zone redundant | 99.99% | 10-100% credit |
| MongoDB Atlas | M10+ | 99.995% | credit |

## Мониторинг SLO

```yaml
# Prometheus recording rules
- record: sli:availability:ratio_30d
  expr: |
    1 - (
      sum(rate(db_errors_total{type="unavailable"}[30d]))
      /
      sum(rate(db_requests_total[30d]))
    )

# Alert: SLO burn rate
- alert: SLOBurnRateTooHigh
  expr: |
    sli:availability:error_rate_1h > (14.4 * (1 - 0.9995))
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Error budget burning too fast (1h window)"
```

### Multi-window burn rate alerts

| Window | Burn rate | Alert |
|--------|-----------|-------|
| 1h | 14.4x | Page (critical) |
| 6h | 6x | Page (warning) |
| 3d | 1x | Ticket |

Если ошибки за последний час потребляют бюджет в 14.4x быстрее нормы → немедленный алерт.
