# Logging

Сбор, хранение и анализ логов инстансов БД.

## Типы логов

| Лог | Описание | Пример |
|-----|----------|--------|
| **DB error log** | Ошибки и предупреждения СУБД | ERROR: relation "users" does not exist |
| **Slow query log** | Запросы медленнее threshold | LOG: duration: 5234.123 ms statement: SELECT ... |
| **Audit log** | Все операции (pgAudit) | AUDIT: user=admin action=DROP TABLE |
| **Connection log** | Подключения/отключения | LOG: connection authorized: user=app database=mydb |
| **Agent log** | Операции агента | INFO: backup completed successfully |
| **Proxy log** | Подключения через proxy | INFO: client connected, routed to primary |

## Конфигурация PostgreSQL

```
# postgresql.conf
log_destination = 'stderr'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d.log'

# Что логировать
log_min_duration_statement = 1000    # slow queries > 1 sec
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0                   # логировать все temp files
log_checkpoints = on
log_autovacuum_min_duration = 0      # логировать autovacuum

# Формат
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
```

## Pipeline сбора логов

```
DB Instance                     Platform
┌───────────┐                  ┌──────────────┐
│ PostgreSQL │──stdout/file──►│  Log Agent    │
│           │                 │  (Fluent Bit) │
└───────────┘                 └──────┬────────┘
                                      │
                              ┌───────┴────────┐
                              │  Log Pipeline  │
                              │  (Vector/Fluentd)│
                              └───────┬────────┘
                                      │
                      ┌───────────────┼───────────────┐
                      │               │               │
               ┌──────┴─────┐ ┌──────┴─────┐ ┌──────┴─────┐
               │ Loki/ES    │ │ S3 Archive │ │ Alerting   │
               │ (search)   │ │ (long-term)│ │ (patterns) │
               └────────────┘ └────────────┘ └────────────┘
```

### Fluent Bit sidecar (Kubernetes)

```yaml
containers:
  - name: postgresql
    image: postgres:16
    volumeMounts:
      - name: logs
        mountPath: /var/log/postgresql
  - name: log-agent
    image: fluent/fluent-bit:latest
    volumeMounts:
      - name: logs
        mountPath: /var/log/postgresql
        readOnly: true
      - name: fluent-bit-config
        mountPath: /fluent-bit/etc/
```

## Structured Logging

Для агента и платформенных компонентов — JSON формат.

```json
{
  "timestamp": "2026-02-11T10:30:00Z",
  "level": "error",
  "component": "agent",
  "instance_id": "inst-abc123",
  "message": "health check failed",
  "error": "connection refused",
  "retry_count": 3
}
```

## Логи для пользователя

### Через API

```
GET /v1/instances/{id}/logs?severity=ERROR&limit=100&since=2026-02-11T00:00:00Z

Response:
{
  "logs": [
    {
      "timestamp": "2026-02-11T10:30:00Z",
      "severity": "ERROR",
      "message": "relation \"users\" does not exist",
      "detail": "..."
    }
  ]
}
```

### Фильтрация

Перед отдачей пользователю — фильтрация sensitive данных:
- Удаление internal IP addresses
- Маскирование паролей в query strings
- Удаление internal component names
- Обфускация путей файловой системы

## Retention

| Tier | Хранилище | Retention | Назначение |
|------|-----------|-----------|------------|
| Hot | Loki / Elasticsearch | 7-30 дней | Поиск и анализ |
| Warm | Compressed in object storage | 90 дней | Troubleshooting |
| Cold | S3 Glacier / Archive | 1-7 лет | Compliance, audit |

## Log-based Alerts

```yaml
# Alert: слишком много ошибок
- alert: HighErrorRate
  expr: |
    sum(rate({job="postgresql", level="ERROR"}[5m])) by (instance_id) > 10
  for: 5m
  labels:
    severity: warning

# Alert: deadlock detected
- alert: DeadlockDetected
  expr: |
    count_over_time({job="postgresql"} |= "deadlock detected" [5m]) > 0
  labels:
    severity: warning
```
