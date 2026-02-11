# High-Level Architecture

Общая архитектура DBaaS-платформы и взаимодействие компонентов.

## Слои

```
┌─────────────────────────────────────────────────────────────┐
│                        Clients                              │
│              UI Console  │  CLI  │  Terraform               │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS
┌──────────────────────────┴──────────────────────────────────┐
│                      API Gateway                            │
│            Rate limiting, Auth, Routing                      │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────┐
│                     Control Plane                           │
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐              │
│  │ Instance  │  │  Backup   │  │ Billing   │              │
│  │ Manager   │  │  Service  │  │ Service   │              │
│  └─────┬─────┘  └─────┬─────┘  └───────────┘              │
│        │              │                                     │
│  ┌─────┴─────┐  ┌─────┴─────┐  ┌───────────┐              │
│  │ Scheduler │  │  Storage  │  │Monitoring │              │
│  │           │  │  Manager  │  │Aggregator │              │
│  └─────┬─────┘  └───────────┘  └───────────┘              │
│        │                                                    │
│  ┌─────┴──────────────────┐                                │
│  │   Metadata Database    │  (PostgreSQL / etcd)            │
│  └────────────────────────┘                                │
└──────────────────────────┬──────────────────────────────────┘
                           │ Agent gRPC / Message Queue
┌──────────────────────────┴──────────────────────────────────┐
│                      Data Plane                             │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Instance A │  │  Instance B │  │  Instance C │        │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │        │
│  │ │ DB (PG) │ │  │ │ DB (PG) │ │  │ │DB (MySQL)│ │        │
│  │ ├─────────┤ │  │ ├─────────┤ │  │ ├─────────┤ │        │
│  │ │  Agent  │ │  │ │  Agent  │ │  │ │  Agent  │ │        │
│  │ ├─────────┤ │  │ ├─────────┤ │  │ ├─────────┤ │        │
│  │ │  Proxy  │ │  │ │  Proxy  │ │  │ │  Proxy  │ │        │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│  Infrastructure: Kubernetes / VMs / Bare Metal              │
└─────────────────────────────────────────────────────────────┘
```

## Основные подсистемы

### Instance Manager
Ядро Control Plane. Управляет жизненным циклом инстансов через state machine.

```
States: Pending → Creating → Running → Modifying → Deleting → Deleted
                                ↕
                            Degraded
```

### Scheduler
Выбирает где разместить инстанс: регион, зона, хост. Учитывает:
- Доступные ресурсы (CPU, RAM, disk)
- Anti-affinity (не размещать реплики на одном хосте)
- Compliance constraints (данные в определённом регионе)
- Cost optimization

### Agent (на каждом инстансе)
- Получает команды от Control Plane (apply config, take backup)
- Отправляет метрики, логи, health status
- Выполняет локальные операции (promote replica, restart)

### Proxy / Connection Pooler
- Маршрутизация подключений (read → replica, write → primary)
- Connection pooling (PgBouncer, ProxySQL)
- TLS termination
- Transparent failover для клиента

## Коммуникация CP ↔ DP

### Pull-модель (agent pulls from CP)

```
Agent → gRPC long-poll → Control Plane
Agent ← commands ← Control Plane
```

- Agent инициирует соединение → проще с сетевой изоляцией
- CP не нужен доступ в сеть инстанса

### Push-модель (CP pushes to agent)

```
Control Plane → gRPC → Agent
```

- Быстрее доставка команд
- CP нужен network path к каждому инстансу

### Message Queue

```
CP → Kafka/NATS → Agent
Agent → Kafka/NATS → CP
```

- Loose coupling, надёжная доставка
- Буферизация при недоступности агента

## Infrastructure Layer

| Подход | Описание | Примеры |
|--------|----------|---------|
| Kubernetes | Pod на инстанс, Operator для управления | CloudNativePG, Zalando |
| VM-based | VM на инстанс, Terraform/Ansible | RDS, Cloud SQL |
| Bare metal | Максимальная производительность | On-premise enterprise |

Kubernetes — наиболее распространённый подход для новых DBaaS благодаря декларативности и экосистеме.
