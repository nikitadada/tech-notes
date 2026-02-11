# Upgrades

Обновление версии СУБД и компонентов платформы.

## Типы обновлений

### Minor Version Upgrade

Например: PostgreSQL 16.2 → 16.4. Обычно содержит bugfixes и security patches.

- Как правило не требует миграции данных
- Restart СУБД необходим
- Низкий риск

### Major Version Upgrade

Например: PostgreSQL 15 → 16. Может содержать несовместимые изменения.

- Требует миграции данных (pg_upgrade или dump/restore)
- Более высокий риск
- Может потребовать изменений в приложении

### Platform Component Upgrade

Обновление Agent, Proxy, OS, Kubernetes.

- Обычно transparent для пользователя
- Rolling update компонентов

## Стратегии обновления

### In-Place Upgrade

Обновление на том же инстансе.

```
1. Stop DB
2. Upgrade binaries
3. Run migration (pg_upgrade)
4. Start DB
5. Validate

Downtime: 1-30 минут (зависит от размера данных)
```

### Blue-Green Upgrade

Создание нового инстанса с новой версией, переключение трафика.

```
1. Create new instance (v16)
2. Setup replication: old (v15) → new (v16)
   (logical replication для cross-version)
3. Wait for sync
4. Stop writes to old
5. Promote new as primary
6. Switch DNS / endpoints
7. Keep old as rollback option
8. Delete old after validation

Downtime: секунды (момент переключения)
```

### Rolling Upgrade (для кластера)

```
Cluster: Primary (v15) + Replica-1 (v15) + Replica-2 (v15)

1. Upgrade Replica-2 → v16
2. Validate Replica-2
3. Upgrade Replica-1 → v16
4. Validate Replica-1
5. Failover: Replica-1 (v16) becomes Primary
6. Upgrade old Primary → v16

Downtime: failover window (~10-30 sec)
```

## Maintenance Windows

Пользователь определяет когда платформа может выполнять обновления.

```json
{
  "maintenance_window": {
    "day": "sunday",
    "start_hour": 3,
    "duration_hours": 4,
    "timezone": "Europe/Moscow"
  }
}
```

- Обязательные security patches — применяются в ближайшее maintenance window
- Optional upgrades — пользователь выбирает когда
- Emergency patches — могут применяться вне окна (с уведомлением)

## Pre-flight Checks

Перед обновлением:

```go
func preflightChecks(instance *Instance, targetVersion string) []Check {
    return []Check{
        checkCompatibility(instance.Engine, instance.Version, targetVersion),
        checkExtensionsCompatibility(instance.Extensions, targetVersion),
        checkReplicationHealth(instance),
        checkDiskSpace(instance), // upgrade может требовать temp space
        checkBackupFreshness(instance), // свежий бэкап перед upgrade
        checkActiveConnections(instance),
        checkRunningQueries(instance), // нет long-running transactions
    }
}
```

## Rollback

### Minor upgrade rollback
- Downgrade бинарников + restart
- Обычно безопасно (data format не меняется)

### Major upgrade rollback
- Сложнее: data format может измениться
- Blue-green: просто переключить обратно на старый инстанс
- In-place: restore из бэкапа, сделанного перед upgrade

**Правило:** всегда делай бэкап перед major upgrade.

## Уведомления

```
[Pending]   → "Доступна версия PostgreSQL 16.4. Обновление запланировано на {maintenance_window}"
[Scheduled] → "Обновление запланировано на 2026-02-15 03:00 UTC"
[Started]   → "Начато обновление инстанса inst-abc123"
[Completed] → "Инстанс inst-abc123 обновлён до PostgreSQL 16.4"
[Failed]    → "Обновление не удалось. Выполнен откат к предыдущей версии"
```
