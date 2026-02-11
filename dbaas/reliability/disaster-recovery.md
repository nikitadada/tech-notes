# Disaster Recovery (DR)

Восстановление после катастрофических отказов: потеря целого региона, массовая коррупция данных, security breach.

## RPO и RTO

| Метрика | Определение | Пример |
|---------|-------------|--------|
| **RPO** (Recovery Point Objective) | Максимально допустимая потеря данных | RPO = 1 час → можно потерять до 1 часа данных |
| **RTO** (Recovery Time Objective) | Время восстановления сервиса | RTO = 30 мин → сервис должен работать через 30 мин |

```
                  RPO                RTO
    ◄───────────────┤     Disaster    ├──────────►
  Last backup     Data lost        Recovery     Service restored
```

## DR стратегии

### 1. Backup & Restore

Самая простая. Бэкапы в другом регионе.

```
Region A (primary): Running instance + WAL archiving → S3 (Region B)

Disaster in Region A:
  1. Create new instance in Region B
  2. Restore from backup
  3. Replay WAL to latest point
  4. Update DNS

RPO: зависит от частоты бэкапов (с WAL → минуты)
RTO: 30 мин — несколько часов (зависит от размера)
```

**Стоимость:** низкая (только storage для бэкапов).

### 2. Warm Standby

Async replica в другом регионе. Всегда актуальна, но read-only.

```
Region A: Primary ──async replication──► Region B: Standby (read-only)

Disaster in Region A:
  1. Promote standby in Region B
  2. Update DNS
  3. Accept writes

RPO: секунды-минуты (replication lag)
RTO: 1-5 минут
```

**Стоимость:** средняя (compute + storage для standby).

### 3. Hot Standby (Active-Active)

Несколько регионов принимают запись. Нужен distributed DB (Spanner, CockroachDB).

```
Region A: Node 1 ←──consensus──→ Region B: Node 2
              ↕                        ↕
         Region C: Node 3

Disaster in Region A:
  Regions B+C продолжают работать (quorum сохранён)

RPO: 0 (synchronous consensus)
RTO: секунды (automatic)
```

**Стоимость:** высокая. Latency записи выше (cross-region consensus).

## DR Plan

### Компоненты плана

```
1. Scope
   - Какие системы покрыты
   - Приоритеты восстановления (Tier 1, 2, 3)

2. Triggers
   - Когда активировать DR
   - Кто принимает решение

3. Procedures
   - Step-by-step для каждого сценария
   - Runbooks для операторов

4. Communication
   - Каналы оповещения
   - Status page обновления

5. Testing
   - Расписание DR drills
   - Acceptance criteria
```

### Тестирование DR

```
Quarterly DR drill:
  1. Simulate region failure
  2. Execute DR procedure
  3. Measure actual RPO and RTO
  4. Validate data integrity
  5. Document findings
  6. Update procedures if needed
```

## Cross-Region Backup Architecture

```
┌─── Region A (primary) ───┐     ┌─── Region B (DR) ──────┐
│                           │     │                         │
│  Instance → WAL archive ──┼────►│  S3 bucket (replicated) │
│             ↓             │     │         ↓               │
│  Daily full backup ───────┼────►│  Backup storage         │
│                           │     │         ↓               │
│                           │     │  [Standby instance]     │
└───────────────────────────┘     └─────────────────────────┘
```

## Сценарии и ответы

| Сценарий | Ответ | RPO | RTO |
|----------|-------|-----|-----|
| Падение Pod/VM | Auto failover (same region) | 0 | 30 сек |
| Потеря AZ | Failover to other AZ | 0 | 1 мин |
| Потеря региона | DR failover to other region | минуты | 5-30 мин |
| Коррупция данных | PITR из backup | зависит от detection time | 30+ мин |
| Accidental DROP TABLE | PITR до момента DROP | минуты | 15-30 мин |
| Ransomware | Restore из immutable backup | часы | часы |
