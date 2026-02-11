# Deletion

Удаление инстанса БД. Необратимая операция — требует предосторожностей.

## API

```json
DELETE /v1/instances/{id}

// Защита от случайного удаления
Headers:
  X-Confirm-Delete: inst-abc123
```

## Flow

```
1. Validate    → проверить permissions, protection lock
2. Stop access → закрыть connections, убрать из DNS
3. Final backup → автоматический бэкап перед удалением (опционально)
4. Stop DB     → graceful shutdown СУБД
5. Cleanup     → удалить compute, network, monitoring
6. Retain data → сохранить storage на grace period
7. Purge       → окончательное удаление данных
8. Release     → освободить квоты, прекратить биллинг
```

## Защита от случайного удаления

### Deletion Protection

```json
PATCH /v1/instances/{id}
{
  "deletion_protection": true
}
```

Пока `deletion_protection = true`, DELETE возвращает ошибку. Нужно сначала явно снять protection.

### Confirmation

```json
DELETE /v1/instances/{id}
{
  "confirm_name": "my-postgres"  // ввести имя инстанса
}
```

### Soft Delete с Grace Period

```
DELETE → state: DELETING
  ↓ (grace period: 7 days)
  Данные сохранены, compute остановлен
  ↓ (7 дней без восстановления)
  state: PURGING → окончательное удаление
```

Во время grace period:
- Compute выключен (не тарифицируется)
- Storage сохранён (тарифицируется по reduced rate)
- Можно восстановить через `POST /v1/instances/{id}/undelete`

## Что удаляется

| Ресурс | Когда удаляется | Восстановимость |
|--------|----------------|-----------------|
| Compute (Pod/VM) | Сразу | Нет (stateless) |
| Network (DNS, LB) | Сразу | Нет |
| Monitoring targets | Сразу | Нет |
| Storage (PV/Disk) | После grace period | Да (до purge) |
| Backups | По retention policy | Да (пока хранятся) |
| Metadata (CP) | Soft delete | Да |
| Audit log | Хранится N дней | Да |

## Backup Retention после удаления

```yaml
deletion:
  grace_period_days: 7
  final_backup: true
  backup_retention_after_delete:
    enabled: true
    days: 30  # бэкапы хранятся 30 дней после удаления
```

Позволяет восстановить данные даже после полного удаления инстанса (создать новый из бэкапа).

## Purge (окончательное удаление)

```go
func purgeInstance(ctx context.Context, instance *Instance) error {
    // 1. Удалить storage volumes
    if err := deleteVolumes(ctx, instance.ID); err != nil {
        return err
    }

    // 2. Удалить бэкапы (если retention expired)
    if err := deleteExpiredBackups(ctx, instance.ID); err != nil {
        return err
    }

    // 3. Зачистить secrets
    if err := deleteSecrets(ctx, instance.ID); err != nil {
        return err
    }

    // 4. Финальная отметка в metadata
    return markPurged(ctx, instance.ID)
}
```

**Важно для compliance:** при purge данные должны быть действительно удалены (не просто помечены). Для зашифрованных данных — crypto shredding (удаление ключа шифрования).

## Cascade effects

При удалении инстанса:
- Read replicas удаляются вместе с primary
- Scheduled backups прекращаются
- Алерты деактивируются
- Billing прекращается (с момента остановки compute)
- DNS записи удаляются
