# Pricing Models

Модели тарификации в DBaaS.

## Основные модели

### 1. Fixed (provisioned)

Фиксированная цена за выбранную конфигурацию. Платишь 24/7 независимо от нагрузки.

```
db.m5.large (2 vCPU, 8 GB RAM)
  $0.192/hour = ~$138/month

+ Storage: $0.115/GB-month
+ Backup: $0.095/GB-month
+ Network egress: $0.09/GB
```

**Плюсы:** предсказуемый счёт, гарантированная производительность.
**Минусы:** платишь за idle, нужно угадать нужный размер.

**Примеры:** AWS RDS, GCP Cloud SQL, Azure Database.

### 2. Pay-per-Use (serverless)

Оплата за фактическое потребление ресурсов.

```
Compute: $0.0667 per ACU-hour (Aurora Serverless)
Storage: $0.10/GB-month
I/O: $0.20 per million requests

Ночью (нет нагрузки): $0 compute
Пик: масштабируется до 64 ACU
```

**Плюсы:** нет платы за idle, автоматическое масштабирование.
**Минусы:** непредсказуемый счёт при высокой нагрузке, cold start.

**Примеры:** Neon, Aurora Serverless v2, ClickHouse Cloud.

### 3. Tiered (фиксированные планы)

Набор предопределённых планов.

```
Hobby:    1 vCPU,  1 GB,  10 GB storage  — $0/month
Starter:  1 vCPU,  2 GB,  25 GB storage  — $15/month
Pro:      2 vCPU,  4 GB,  50 GB storage  — $50/month
Business: 4 vCPU,  8 GB, 100 GB storage  — $150/month
```

**Плюсы:** простой выбор, предсказуемая цена.
**Минусы:** ступенчатый рост, может быть overprovisioned.

**Примеры:** Supabase, PlanetScale, Render.

### 4. Credits / Prepaid

Покупка кредитов заранее со скидкой.

```
$1000 credits → 10% discount → $900
Credits consumed based on actual usage
Alert when balance < $100
```

**Примеры:** AWS Reserved Instances (1-3 года commit), GCP Committed Use.

## Компоненты стоимости

```
Monthly bill:
  ├── Compute        $138.00  (2 vCPU × 730h × $0.0946)
  ├── Storage         $11.50  (100 GB × $0.115)
  ├── Backup           $4.75  (50 GB × $0.095)
  ├── Network          $9.00  (100 GB egress × $0.09)
  ├── IOPS             $0.00  (included in general purpose)
  └── HA replica     $138.00  (same as primary)
  ──────────────────────────
  Total:             $301.25
```

## Billing Cycle

```
Usage collection (hourly)
    │
    ▼
Aggregation (daily)
    │
    ▼
Rating (apply prices)
    │
    ▼
Invoice generation (monthly)
    │
    ▼
Payment processing
    │
    ▼
Receipt
```

## Cost Optimization

### Right-sizing

```
Current: 4 vCPU, 16 GB RAM — avg CPU usage 15%
Recommendation: 2 vCPU, 8 GB RAM — save 50%
```

Платформа анализирует метрики и рекомендует оптимальный размер.

### Reserved capacity

```
On-demand:  $0.192/hour
1-year RI:  $0.120/hour (37% savings)
3-year RI:  $0.076/hour (60% savings)
```

### Auto-pause (serverless)

```
No connections for 5 minutes → scale to 0
First connection → cold start (1-5 sec) → scale up

Savings: ~70% for dev/staging with sporadic usage
```

### Storage optimization

```
- Удаление неиспользуемых бэкапов
- Сжатие данных (pg_repack, TOAST)
- Перенос cold data в дешёвое хранилище
- Оптимизация WAL retention
```

## Отображение для пользователя

```json
GET /v1/projects/{id}/billing/current

{
  "period": "2026-02",
  "status": "in_progress",
  "total": 245.67,
  "currency": "USD",
  "breakdown": [
    {"resource": "compute", "amount": 138.00, "details": "2 vCPU × 730h"},
    {"resource": "storage", "amount": 11.50, "details": "100 GB"},
    {"resource": "backup",  "amount": 4.75,  "details": "50 GB"},
    {"resource": "network", "amount": 3.42,  "details": "38 GB egress"},
    {"resource": "ha_replica", "amount": 88.00, "details": "1 replica"}
  ],
  "forecast": {
    "end_of_month": 310.00
  }
}
```
