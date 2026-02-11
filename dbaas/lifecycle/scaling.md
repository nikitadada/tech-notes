# Scaling

Изменение ресурсов инстанса для соответствия нагрузке.

## Vertical Scaling (Scale Up/Down)

Изменение CPU, RAM, disk одного инстанса.

### CPU / Memory

```json
PATCH /v1/instances/{id}
{
  "resources": {
    "cpu": 4,       // было 2
    "memory_gb": 8  // было 4
  }
}
```

**Реализация:**
- В Kubernetes: обновление resource requests/limits Pod'а → recreate Pod
- В VM: resize VM (обычно требует restart)
- Downtime зависит от платформы

**Zero-downtime vertical scaling:**

```
1. Создать новый Pod/VM с новыми ресурсами
2. Настроить репликацию со старого на новый
3. Подождать sync
4. Promote новый как primary
5. Удалить старый

Итого: switchover downtime ~5-30 секунд
```

### Storage

```json
PATCH /v1/instances/{id}
{
  "resources": {
    "storage_gb": 100  // было 50
  }
}
```

- Расширение обычно online (EBS, PD поддерживают resize без downtime)
- Уменьшение обычно не поддерживается
- Filesystem resize: `resize2fs` / `xfs_growfs` — online операция

## Horizontal Scaling

### Read Replicas

Добавление реплик для масштабирования чтения.

```json
POST /v1/instances/{id}/replicas
{
  "count": 2,
  "resources": {
    "cpu": 2,
    "memory_gb": 4
  }
}
```

```
Client reads  → Load Balancer → Replica 1
                              → Replica 2
Client writes → Primary
```

**Важно:**
- Replication lag — реплики могут отставать
- Read-after-write inconsistency — запись на primary, чтение с реплики
- Каждая реплика — дополнительная стоимость

### Connection Pooling

Не масштабирование БД, но масштабирование подключений.

```
1000 app connections → PgBouncer (100 connections) → PostgreSQL
```

PgBouncer в transaction mode: одно соединение к PG обслуживает много клиентов.

## Auto-scaling

Автоматическое изменение ресурсов на основе метрик.

```yaml
autoscaling:
  enabled: true
  storage:
    enabled: true
    threshold_percent: 80      # при 80% заполнении
    increment_gb: 10           # добавить 10GB
    max_gb: 500                # не больше 500GB
  compute:
    enabled: true
    cpu_threshold_percent: 80  # при 80% CPU
    scale_up_cooldown: 5m      # не чаще раз в 5 минут
    scale_down_cooldown: 30m
    min_cpu: 1
    max_cpu: 16
```

### Storage auto-scaling

Наиболее распространённый тип. Диск заканчивается → БД останавливается.

```
Monitor: disk_usage > 80%
  → Trigger: expand storage by 20%
  → Execute: resize PV + filesystem
  → Alert: notify user
```

### Compute auto-scaling

Сложнее: изменение CPU/RAM обычно требует рестарта.

**Подходы:**
- Serverless engines (Neon, Aurora Serverless) — масштабируются прозрачно
- Scheduled scaling — увеличение ресурсов по расписанию (peak hours)
- Reactive scaling — по метрикам, с cooldown и maintenance window

## Serverless Scaling

Отделение compute от storage. Compute масштабируется от 0 до N.

```
             Нет запросов          Пиковая нагрузка
                 │                       │
Compute:     [0 ACU]               [64 ACU]
Storage:     [сохраняется]         [сохраняется]
Cost:        [$0 compute]          [$$ compute]
```

- **ACU (Aurora Capacity Units)** / **CU (Compute Units)** — единица масштабирования
- Scale-to-zero — нет нагрузки → нет расходов на compute
- Cold start — первый запрос после scale-to-zero медленнее (1-5 сек)

**Примеры:** Neon, Aurora Serverless v2, ClickHouse Cloud.

## Ограничения

- Vertical scaling имеет потолок (максимальный размер VM/Pod)
- Horizontal read replicas не масштабируют запись
- Для масштабирования записи нужен sharding (см. [sharding](../../distributed-systems/sharding.md))
- Auto-scaling compute может вызвать кратковременный downtime
