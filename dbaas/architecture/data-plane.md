# Data Plane

Плоскость данных — инфраструктура, на которой работают пользовательские БД и обрабатываются SQL-запросы.

## Компоненты инстанса

```
┌──────────────────── Instance ────────────────────┐
│                                                   │
│  ┌─────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  Proxy  │  │ DB Engine│  │    Agent       │  │
│  │(PgBouncer│  │(PostgreSQL│  │ - health check│  │
│  │ /envoy) │──│  /MySQL)  │  │ - config sync │  │
│  └────┬────┘  └────┬─────┘  │ - backup exec │  │
│       │            │         │ - metrics     │  │
│       │            │         └───────┬────────┘  │
│       │       ┌────┴─────┐          │            │
│       │       │ Storage  │          │            │
│       │       │(local/EBS)│          │            │
│       │       └──────────┘          │            │
└───────┼─────────────────────────────┼────────────┘
        │                             │
   User traffic                  CP communication
```

### DB Engine

Сама СУБД. В зависимости от платформы:
- PostgreSQL, MySQL, MongoDB, Redis и т.д.
- Конфигурируется через Agent (параметры, extensions, users)
- Запущена как процесс в контейнере или на VM

### Agent (sidecar)

Связующее звено между инстансом и Control Plane.

```go
type Agent struct {
    instanceID string
    cpClient   ControlPlaneClient
    dbClient   DatabaseClient
    ticker     *time.Ticker
}

func (a *Agent) Run(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)

    // Heartbeat: отправляет status в CP
    g.Go(func() error { return a.heartbeatLoop(ctx) })

    // Command receiver: получает и выполняет команды от CP
    g.Go(func() error { return a.commandLoop(ctx) })

    // Metrics collector: собирает и отправляет метрики
    g.Go(func() error { return a.metricsLoop(ctx) })

    // Config watcher: применяет изменения конфигурации
    g.Go(func() error { return a.configWatchLoop(ctx) })

    return g.Wait()
}
```

**Обязанности Agent:**
- Heartbeat / health reporting
- Приём и выполнение команд (backup, config change, promote)
- Сбор и отправка метрик
- Управление локальными процессами (restart, failover)

### Proxy / Connection Pooler

Промежуточный слой между клиентом и БД.

| Компонент | Назначение | Пример |
|-----------|-----------|--------|
| PgBouncer | Connection pooling для PostgreSQL | Режимы: session, transaction, statement |
| Odyssey | Connection pooler (Yandex) | Multi-threaded, transaction pooling |
| ProxySQL | Proxy для MySQL | Read/write splitting, query routing |
| Envoy | L4/L7 proxy | Universal, extensible |

**Функции proxy:**
- Connection pooling — переиспользование соединений к БД
- Read/write splitting — read → replica, write → primary
- TLS termination
- Transparent failover — клиент не замечает переключения

### Storage

| Тип | Описание | Применение |
|-----|----------|------------|
| Local SSD | Наименьшая latency | High-performance instances |
| Network block (EBS, PD) | Отделяем от compute | Стандартные instances, snapshots |
| Distributed (Ceph, MinIO) | Self-hosted, scalable | Kubernetes-based platforms |
| Object storage (S3) | Дёшево, для архивов | Бэкапы, WAL archive |

## Deployment Models

### Kubernetes-based

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: user-instance-123
spec:
  instances: 3
  storage:
    size: 50Gi
    storageClass: fast-ssd
  resources:
    requests:
      cpu: "2"
      memory: "4Gi"
  postgresql:
    parameters:
      shared_buffers: "1GB"
      max_connections: "200"
```

Каждый инстанс — Pod (или StatefulSet) с DB + Agent + Proxy containers.

### VM-based

Каждый инстанс — отдельная VM. Terraform/Ansible для provisioning.

```
VM (e2-standard-4)
├── PostgreSQL (systemd service)
├── Agent (systemd service)
├── PgBouncer (systemd service)
├── node_exporter
└── Data disk (/dev/sdb → /var/lib/postgresql)
```

## Network Architecture

```
User → Load Balancer → Proxy → DB Engine
                         │
                    (в пределах Pod/VM)
```

- Каждый инстанс получает уникальный DNS: `inst-123.db.example.com`
- TLS обязателен для внешних подключений
- Private networking для репликации между primary и replicas
