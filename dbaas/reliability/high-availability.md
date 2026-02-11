# High Availability

Обеспечение непрерывной работы БД при отказах компонентов.

## Архитектура HA

```
            ┌────────────────┐
            │  Load Balancer │
            └───────┬────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
   ┌────┴────┐ ┌───┴─────┐ ┌──┴──────┐
   │ Primary │ │Replica 1│ │Replica 2│
   │ (zone-a)│ │(zone-b) │ │(zone-c) │
   └────┬────┘ └─────────┘ └─────────┘
        │       ▲         ▲
        └───────┴─────────┘
         Streaming replication
```

- Primary принимает все записи
- Replicas — synchronous или asynchronous streaming replication
- При отказе primary → promote одну из реплик

## Multi-AZ Deployment

Инстанс распределён по нескольким Availability Zones.

```
AZ-a: Primary
AZ-b: Sync Replica    ← гарантия: данные есть минимум в 2 зонах
AZ-c: Async Replica   ← для масштабирования чтения
```

**Sync replica:** primary ждёт подтверждения от sync replica перед ACK клиенту. Нулевая потеря данных (RPO = 0), но выше latency.

**Async replica:** primary не ждёт. Ниже latency, но при failover возможна потеря последних транзакций.

## Компоненты HA в DBaaS

### Health Checker

```go
type HealthCheck struct {
    Type     string        // tcp, pg_isready, query
    Interval time.Duration // 5s
    Timeout  time.Duration // 3s
    Retries  int           // 3 failures before unhealthy
}

// Типы проверок
// 1. TCP connect — порт доступен
// 2. pg_isready — PostgreSQL принимает соединения
// 3. SELECT 1 — БД отвечает на запросы
// 4. Replication lag — replica не отстаёт критично
```

### Connection Routing

```
Client → DNS (inst.db.example.com)
         │
         ├── Primary endpoint → write queries
         └── Replica endpoint → read queries

При failover: DNS обновляется → новый primary
TTL DNS записи: 5-30 секунд
```

### Proxy-based routing

```
Client → HAProxy / Envoy / PgBouncer
           │
           ├── health check: primary alive? → route writes
           └── health check: replica alive? → route reads

При failover: proxy автоматически переключает
Downtime: 0-5 секунд
```

## Уровни HA

| Уровень | Описание | Downtime / год |
|---------|----------|---------------|
| Нет HA | Single instance | Часы-дни |
| Same-zone replica | Replica в той же AZ | ~1 час |
| Multi-AZ | Replicas в разных AZ | ~1 минута |
| Multi-region | Replicas в разных регионах | ~секунды |

## Availability целевые значения

| SLA | Допустимый downtime/год | Допустимый downtime/месяц |
|-----|------------------------|--------------------------|
| 99% | 3.65 дня | 7.3 часа |
| 99.9% | 8.76 часа | 43.8 минуты |
| 99.95% | 4.38 часа | 21.9 минуты |
| 99.99% | 52.6 минуты | 4.38 минуты |
| 99.999% | 5.26 минуты | 26.3 секунды |

## Что влияет на availability

| Событие | Без HA | С HA (multi-AZ) |
|---------|--------|-----------------|
| Падение процесса БД | Downtime до restart | Failover ~30 сек |
| Отказ VM/Pod | Downtime до reschedule | Failover ~30 сек |
| Отказ зоны | Длительный downtime | Failover ~1 мин |
| Плановое обновление | Downtime 1-5 мин | Rolling upgrade, ~10 сек |
| Storage failure | Restore из backup | Failover + rebuild replica |
