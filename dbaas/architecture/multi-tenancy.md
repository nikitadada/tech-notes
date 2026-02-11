# Multi-Tenancy

Обслуживание множества изолированных пользователей (tenants) на одной платформе.

## Модели multi-tenancy

### 1. Shared Everything

Все tenants используют одну БД, разделены на уровне строк.

```
┌──────────────────────────────┐
│         Один DB Instance     │
│  ┌────────────────────────┐  │
│  │ table: data            │  │
│  │ tenant_id | key | val  │  │
│  │ tenant-A  | ...  | ... │  │
│  │ tenant-B  | ...  | ... │  │
│  │ tenant-C  | ...  | ... │  │
│  └────────────────────────┘  │
└──────────────────────────────┘
```

- Максимальная плотность
- Минимальная изоляция
- Сложно масштабировать отдельного tenant'а
- Используется для metadata DB самого Control Plane

### 2. Shared Instance, Separate Schema/Database

Один DB instance, но каждый tenant в своей schema или database.

```
┌──────────────────────────────┐
│         Один DB Instance     │
│  ┌──────┐ ┌──────┐ ┌──────┐ │
│  │ DB A │ │ DB B │ │ DB C │ │
│  └──────┘ └──────┘ └──────┘ │
└──────────────────────────────┘
```

- Лучше изоляция (separate permissions)
- Shared CPU/RAM/IO
- Средняя плотность

### 3. Shared Host, Separate Instance

Несколько DB instance на одном хосте.

```
┌──────────── Host ────────────┐
│ ┌────────┐ ┌────────┐       │
│ │ PG (A) │ │ PG (B) │ ...   │
│ └────────┘ └────────┘       │
│      Shared OS / VM          │
└──────────────────────────────┘
```

- Process-level изоляция
- cgroups для ограничения ресурсов
- Noisy neighbor possible

### 4. Dedicated Instance (один инстанс на tenant)

Самая распространённая модель в production DBaaS.

```
┌─── Host 1 ───┐  ┌─── Host 2 ───┐
│ ┌──────────┐  │  │ ┌──────────┐  │
│ │ PG (A)   │  │  │ │ PG (B)   │  │
│ └──────────┘  │  │ └──────────┘  │
└───────────────┘  └───────────────┘
```

- Полная изоляция ресурсов
- Независимое масштабирование
- Дороже, но предсказуемо

## Изоляция в Kubernetes

```yaml
# Namespace per tenant
apiVersion: v1
kind: Namespace
metadata:
  name: tenant-abc123
  labels:
    tenant-id: abc123
---
# Resource quota per tenant
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-abc123
spec:
  hard:
    requests.cpu: "8"
    requests.memory: "16Gi"
    persistentvolumeclaims: "10"
---
# Network policy: isolate tenant namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-tenant
  namespace: tenant-abc123
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              tenant-id: abc123
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              tenant-id: abc123
```

## Tenant routing

### По DNS

```
tenant-a.db.example.com → Instance A (10.0.1.5)
tenant-b.db.example.com → Instance B (10.0.2.8)
```

### По SNI (Server Name Indication)

TLS proxy определяет tenant по SNI и маршрутизирует к нужному инстансу.

### По auth credentials

Proxy определяет tenant по username/password и направляет к нужному инстансу.

## Noisy Neighbor Problem

Один tenant потребляет непропорционально много ресурсов, влияя на других.

**Mitigation:**
- cgroups / Kubernetes resource limits
- I/O throttling на уровне storage
- Network bandwidth limits
- Rate limiting на API уровне
- Автоматический алерт + throttle при превышении

## Данные Control Plane

CP сам multi-tenant: хранит данные всех пользователей.

```sql
-- Всё привязано к project_id
SELECT * FROM instances WHERE project_id = 'proj-123';
SELECT * FROM backups WHERE project_id = 'proj-123';

-- Row-Level Security (PostgreSQL)
CREATE POLICY tenant_isolation ON instances
    USING (project_id = current_setting('app.current_project'));
```
