# Network Isolation

Ограничение сетевого доступа к инстансам БД.

## Уровни изоляции

### Public Access (не рекомендуется для production)

Инстанс доступен из интернета по публичному IP.

```
Internet → Public IP → Instance
```

Защита: IP allowlist + TLS + strong auth.

### IP Allowlist

```json
PATCH /v1/instances/{id}
{
  "network": {
    "allowed_cidrs": [
      "10.0.0.0/8",          // internal VPC
      "203.0.113.10/32"      // office IP
    ]
  }
}
```

Реализация: Security Groups (AWS), Firewall Rules (GCP), NSG (Azure), NetworkPolicy (K8s).

### Private Networking (VPC Peering)

Инстанс доступен только из VPC пользователя.

```
┌─── User VPC (10.0.0.0/16) ───┐     ┌─── DBaaS VPC (172.16.0.0/16) ───┐
│                                │     │                                   │
│  App Server (10.0.1.5)        │◄───►│  DB Instance (172.16.5.10)        │
│                                │     │                                   │
└────────────────────────────────┘     └───────────────────────────────────┘
                    VPC Peering
```

- Трафик не выходит в интернет
- Нужна настройка peering + route tables
- CIDR не должны пересекаться

### Private Link / Private Endpoint

Более безопасно чем VPC Peering: однонаправленное соединение.

```
┌─── User VPC ───┐          ┌─── DBaaS VPC ───┐
│                 │          │                   │
│  App ──► Private Endpoint ──► NLB ──► Instance│
│                 │          │                   │
└─────────────────┘          └───────────────────┘
```

- Трафик через cloud backbone (не интернет)
- Unidirectional: пользователь → DBaaS (DBaaS не видит User VPC)
- Нет необходимости в CIDR coordination
- AWS PrivateLink, GCP Private Service Connect, Azure Private Link

### Kubernetes NetworkPolicy

Для инстансов внутри K8s кластера.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-instance-policy
  namespace: tenant-abc
spec:
  podSelector:
    matchLabels:
      app: postgresql
      instance-id: inst-abc123
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Только от proxy
    - from:
        - podSelector:
            matchLabels:
              role: proxy
      ports:
        - port: 5432
  egress:
    # Replication к другим DB pods
    - to:
        - podSelector:
            matchLabels:
              instance-id: inst-abc123
      ports:
        - port: 5432
    # Backup to S3 (через NAT gateway)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - port: 443
```

## Segmentation

### Instance-level

Каждый инстанс изолирован от других. Никакой cross-instance трафик.

### Tenant-level

Все инстансы tenant'а в одном network segment, изолированы от других tenant'ов.

### Management plane

Отдельная сеть для управляющего трафика (agent ↔ control plane), недоступная для пользователей.

```
┌── Management Network ──┐
│  Agent ←→ Control Plane │
└─────────────────────────┘
         ↕ (isolated)
┌── Data Network ─────────┐
│  Client ←→ DB Instance  │
└─────────────────────────┘
```

## DNS Security

```
inst-abc123.db.example.com → private IP

- Private DNS zones: резолвятся только внутри VPC
- Split-horizon DNS: разные ответы для internal/external
- DNSSEC: защита от подмены DNS записей
```
