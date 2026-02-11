# What is DBaaS

Database as a Service — управляемый сервис баз данных, где провайдер берёт на себя инфраструктуру, а пользователь работает только с данными.

## Что берёт на себя провайдер

| Ответственность | Self-managed | DBaaS |
|-----------------|:-----------:|:-----:|
| Железо / VM | Вы | Провайдер |
| Установка СУБД | Вы | Провайдер |
| Патчи и обновления | Вы | Провайдер |
| Бэкапы | Вы | Провайдер |
| High Availability | Вы | Провайдер |
| Мониторинг инфраструктуры | Вы | Провайдер |
| Схема данных | Вы | Вы |
| Запросы и индексы | Вы | Вы |
| Логика приложения | Вы | Вы |

## Shared Responsibility Model

```
┌──────────────────────────────────────┐
│          Пользователь                │
│  Схема, запросы, индексы, доступы    │
├──────────────────────────────────────┤
│          DBaaS Platform              │
│  Provisioning, HA, бэкапы, патчи    │
├──────────────────────────────────────┤
│          Infrastructure              │
│  Compute, Storage, Network           │
└──────────────────────────────────────┘
```

## Ключевые возможности DBaaS

- **Self-service provisioning** — создание инстанса через API/UI за минуты
- **Автоматические бэкапы** — по расписанию + point-in-time recovery
- **High Availability** — автоматический failover при отказе
- **Масштабирование** — vertical (CPU/RAM) и horizontal (read replicas, sharding)
- **Мониторинг** — встроенные метрики, алерты, slow query log
- **Обновления** — zero-downtime upgrades управляемые платформой
- **Биллинг** — pay-per-use, прозрачная тарификация

## Типы DBaaS

### По модели управления

| Тип | Описание | Пример |
|-----|----------|--------|
| Fully managed | Провайдер управляет всем | Amazon RDS, Cloud SQL |
| Semi-managed | Пользователь контролирует часть параметров | Crunchy Bridge |
| Self-hosted platform | Платформа работает в вашем кластере | Percona Operator, Zalando Postgres Operator |

### По типу СУБД

| Тип | Примеры СУБД | Примеры сервисов |
|-----|-------------|-----------------|
| Relational | PostgreSQL, MySQL | RDS, Cloud SQL, AlloyDB |
| Document | MongoDB | Atlas, DocumentDB |
| Key-Value | Redis, DynamoDB | ElastiCache, MemoryDB |
| Time-series | ClickHouse, TimescaleDB | Managed ClickHouse, Timescale Cloud |
| Graph | Neo4j | AuraDB |

## Когда DBaaS vs Self-managed

**DBaaS подходит:**
- Нет DBA в команде
- Нужно быстро стартовать
- Стандартные требования к БД
- Готовы платить за удобство

**Self-managed подходит:**
- Специфичные требования к конфигурации
- Compliance запрещает managed services
- Нужен полный контроль над железом
- Экономия при большом масштабе
