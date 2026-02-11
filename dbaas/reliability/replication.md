# Replication в DBaaS

Копирование данных между инстансами для HA, масштабирования чтения и disaster recovery.

## Streaming Replication (PostgreSQL)

Primary отправляет WAL-записи replica'м по TCP-соединению в реальном времени.

```
Primary ──WAL stream──► Standby 1
                     ──► Standby 2
```

### Synchronous

```sql
-- postgresql.conf на primary
synchronous_standby_names = 'FIRST 1 (standby1, standby2)'
synchronous_commit = on
```

Primary ждёт подтверждения от минимум одного sync standby.

| `synchronous_commit` | Поведение |
|----------------------|-----------|
| `off` | Не ждать flush WAL даже на primary |
| `local` | Ждать flush на primary |
| `remote_write` | Ждать write (не flush) на standby |
| `on` | Ждать flush на standby (default) |
| `remote_apply` | Ждать apply на standby (read-your-writes на replica) |

### Asynchronous

Primary не ждёт подтверждения. Минимальное влияние на latency записи.

**Replication lag:** время отставания replica от primary. Зависит от нагрузки, сети, производительности replica.

```sql
-- Проверка lag на primary
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

## Logical Replication

Репликация на уровне логических изменений (INSERT, UPDATE, DELETE). Не побайтовая копия WAL.

```sql
-- Publisher (primary)
CREATE PUBLICATION my_pub FOR TABLE users, orders;

-- Subscriber (replica)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=primary dbname=mydb'
    PUBLICATION my_pub;
```

**Применения в DBaaS:**
- Cross-version replication (для major upgrade)
- Selective replication (только определённые таблицы)
- Cross-region replication (разные кластеры)
- Data migration между инстансами

## Топологии

### Primary-Replica

```
Primary → Replica 1
        → Replica 2
```

Стандартная модель. Write → primary, read → любой.

### Cascading Replication

```
Primary → Replica 1 → Replica 3
                    → Replica 4
        → Replica 2
```

Снижает нагрузку на primary (не нужно отправлять WAL всем). Replica 1 "переотправляет" WAL дальше.

### Multi-Region

```
Region A: Primary → Replica (sync, same region)
              ↓
Region B: Replica (async, cross-region) → read-only endpoint
```

Cross-region replica для DR и low-latency reads в другом регионе.

## Мониторинг репликации

Ключевые метрики:

| Метрика | Описание | Alert threshold |
|---------|----------|-----------------|
| `replication_lag_seconds` | Отставание в секундах | > 30s |
| `replication_lag_bytes` | Отставание в байтах WAL | > 100MB |
| `replication_state` | streaming / catchup / stopped | != streaming |
| `wal_receiver_status` | Состояние приёма WAL | != streaming |

```yaml
# Alerting rule
- alert: HighReplicationLag
  expr: pg_replication_lag_seconds > 30
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Replication lag > 30s on {{ $labels.instance }}"
```

## Replication Slots

Гарантируют, что primary не удалит WAL, пока replica его не получила.

```sql
-- Создание
SELECT pg_create_physical_replication_slot('replica1_slot');

-- Мониторинг (WAL retention)
SELECT slot_name, active, pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
FROM pg_replication_slots;
```

**Опасность:** неактивный slot → WAL накапливается → диск заканчивается.

В DBaaS: автоматическое удаление неактивных slots или `max_slot_wal_keep_size`.
