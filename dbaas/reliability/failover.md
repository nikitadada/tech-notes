# Failover

Автоматическое переключение с неработающего primary на healthy replica.

## Failover Flow

```
1. Detection   → primary unavailable (health checks fail)
2. Validation  → подтверждение: fencing, multiple checks
3. Election    → выбор лучшей реплики
4. Promotion   → promote replica to primary
5. Routing     → обновить DNS / proxy routing
6. Notification → alert пользователю
7. Rebuild     → создать новую реплику взамен старого primary
```

## Detection

### Failure detection

```
Health Checker → pg_isready every 5s
  3 consecutive failures (15s)
    → Instance marked UNHEALTHY
    → Confirm from multiple monitors (avoid false positive)
    → Trigger failover
```

**Проблема false positives:**
- Сетевой partition — primary жив, но checker не видит
- Высокая нагрузка — primary медленно отвечает
- GC pause — временная пауза

**Решения:**
- Консенсус между несколькими мониторами
- Quorum-based detection (большинство мониторов согласны)
- Увеличенные таймауты перед failover

## Fencing (изоляция старого primary)

Критично: два primary одновременно = split brain = потеря данных.

```go
func fenceOldPrimary(ctx context.Context, instance *Instance) error {
    // 1. Revoke network access (security group / network policy)
    if err := blockNetworkAccess(ctx, instance.ID); err != nil {
        return err
    }

    // 2. Shutdown old primary process
    if err := shutdownDB(ctx, instance.ID); err != nil {
        // Если не можем — STONITH (Shoot The Other Node In The Head)
        return forceKillNode(ctx, instance.ID)
    }

    return nil
}
```

**STONITH:** принудительное выключение узла (power off VM, delete Pod) если graceful shutdown невозможен.

## Election (выбор новой primary)

Критерии выбора реплики для promotion:

```go
type ReplicaCandidate struct {
    ID             string
    ReplicationLag time.Duration
    WALPosition    uint64
    Zone           string
    IsSync         bool
}

func selectBestReplica(candidates []ReplicaCandidate) *ReplicaCandidate {
    // 1. Приоритет: synchronous replica (нет data loss)
    // 2. Минимальный replication lag
    // 3. Самая свежая WAL position
    // 4. Предпочтение другой зоне (для diversity)
    sort.Slice(candidates, func(i, j int) bool {
        if candidates[i].IsSync != candidates[j].IsSync {
            return candidates[i].IsSync
        }
        return candidates[i].WALPosition > candidates[j].WALPosition
    })
    return &candidates[0]
}
```

## Promotion

```sql
-- PostgreSQL: promote replica to primary
SELECT pg_promote();

-- Или через файл
touch /var/lib/postgresql/data/promote
```

После promotion:
- Replica перестаёт принимать WAL от старого primary
- Начинает принимать write-запросы
- Остальные replicas переподключаются к новому primary

## Routing Update

```
DNS-based:
  inst-abc123.db.example.com → old primary IP
  (update) → new primary IP
  TTL: 5 seconds → клиенты переключатся за 5-10 сек

Proxy-based:
  HAProxy health check → detects new primary
  Route update → immediate
  Downtime: < 1 second
```

## Timeline failover

```
T+0s    Primary fails
T+15s   Detection (3 × 5s health checks)
T+16s   Validation (confirm from multiple sources)
T+17s   Fencing old primary
T+18s   Promote replica
T+20s   Update routing (DNS/proxy)
T+25s   New primary accepting connections
─────────────────────────────────────
Total:  ~25 seconds
```

## Типы failover

| Тип | Описание | Data loss |
|-----|----------|-----------|
| Planned | Maintenance, upgrade. Graceful switchover | Нет |
| Automatic | Незапланированный отказ, автоматика | Возможна (async replica) |
| Manual | Оператор инициирует через API/UI | Нет (контролируемо) |

## Switchover (planned failover)

```
1. Stop writes to current primary
2. Wait for replicas to catch up (lag = 0)
3. Promote chosen replica
4. Demote old primary to replica
5. Update routing

Data loss: 0 (контролируемый процесс)
Downtime: 5-30 seconds
```

## Тестирование failover

Регулярное тестирование — обязательно.

```
Chaos engineering:
  - Kill primary Pod/VM
  - Network partition (block traffic)
  - Disk full simulation
  - CPU stress test

Validate:
  - Failover time < SLA threshold
  - No data loss (for sync replication)
  - Applications reconnect automatically
  - Monitoring detects and alerts
```
