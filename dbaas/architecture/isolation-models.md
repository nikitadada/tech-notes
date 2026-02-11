# Isolation Models

Уровни изоляции между tenant'ами в DBaaS. От логической до полной физической.

## Уровни изоляции

```
              Изоляция ↑              Стоимость ↑
              ─────────────────────────────────────
    Level 5 │ Dedicated Infrastructure (account)
    Level 4 │ Dedicated Host (bare metal / VM)
    Level 3 │ Dedicated Pod/Container (namespace)
    Level 2 │ Shared Instance, Separate DB/Schema
    Level 1 │ Shared Everything (row-level)
              ─────────────────────────────────────
              Плотность ↑             Простота ↑
```

## Compute Isolation

### Shared process (Level 1-2)

Один процесс СУБД обслуживает нескольких tenant'ов.

```
PostgreSQL process
  ├── database: tenant_a
  ├── database: tenant_b
  └── database: tenant_c
```

- Общий buffer pool, WAL, connections
- Один tenant с heavy query тормозит остальных
- Изоляция через RBAC и schemas

### Container isolation (Level 3)

Отдельный контейнер/Pod на каждый инстанс.

```
Node
  ├── Pod (tenant-a): PostgreSQL + Agent
  ├── Pod (tenant-b): PostgreSQL + Agent
  └── Pod (tenant-c): PostgreSQL + Agent
```

- cgroups ограничивают CPU/RAM
- Отдельный PID namespace
- Shared kernel — потенциальные kernel exploits

### VM isolation (Level 4)

Отдельная VM на каждый инстанс.

- Аппаратная изоляция через hypervisor
- Отдельный kernel
- Стандарт для compliance-sensitive workloads

### Dedicated infrastructure (Level 5)

Выделенные физические серверы, отдельный аккаунт облака.

- Максимальная изоляция
- Для крупных enterprise клиентов
- Значительная стоимость

## Storage Isolation

| Модель | Описание | Изоляция |
|--------|----------|----------|
| Shared volume | Один PV, разделение через директории | Низкая |
| Separate PV | Отдельный persistent volume на инстанс | Средняя |
| Separate disk | Выделенный физический диск / EBS | Высокая |
| Encrypted per-tenant | Шифрование отдельным ключом | Высокая + crypto |

```yaml
# Separate PV per instance (Kubernetes)
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: encrypted-ssd
      resources:
        requests:
          storage: 50Gi
```

## Network Isolation

### Level 1: Shared network, port-based

Все инстансы в одной сети, разделение по портам.

### Level 2: Network Policy

```yaml
# Kubernetes NetworkPolicy: только свой namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-tenant
spec:
  podSelector: {}
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tenant-id: abc123
```

### Level 3: Separate VPC / VLAN

Каждый tenant в своей виртуальной сети.

```
VPC tenant-a (10.1.0.0/16)
  └── Instance A

VPC tenant-b (10.2.0.0/16)
  └── Instance B

# Связь через VPC peering или private endpoint
```

### Level 4: Private Link / Endpoint

```
Tenant VPC ──── Private Endpoint ──── DBaaS VPC
                (no public internet)
```

AWS PrivateLink, GCP Private Service Connect, Azure Private Link.

## Secrets Isolation

| Модель | Описание |
|--------|----------|
| Shared KMS key | Все tenant'ы шифруются одним ключом |
| Per-tenant KMS key | Каждый tenant — отдельный ключ (envelope encryption) |
| Customer-managed key (CMEK) | Tenant управляет своим ключом |
| BYOK (Bring Your Own Key) | Tenant приносит ключ из своего KMS |

## Выбор модели

| Tier | Compute | Storage | Network | Типичная цена |
|------|---------|---------|---------|---------------|
| Free | Shared container | Shared PV | Shared | $0 |
| Standard | Dedicated container | Separate PV | Network Policy | $$ |
| Business | Dedicated VM | Dedicated disk | Separate VPC | $$$ |
| Enterprise | Dedicated host | Encrypted per-tenant | Private Link | $$$$ |

Большинство DBaaS предлагают несколько tier'ов, позволяя пользователю выбрать баланс изоляции и стоимости.
