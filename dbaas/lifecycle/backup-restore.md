# Backup & Restore

Бэкапы — критическая функция DBaaS. Защита от потери данных, human error, коррупции.

## Типы бэкапов

### Physical Backup (Full)

Копия файлов данных на диске.

```
pg_basebackup → tar/dir → S3/GCS
```

- Быстрый restore (копируем файлы обратно)
- Большой размер
- Привязан к версии СУБД

### Logical Backup

Дамп SQL-команд (pg_dump, mysqldump).

```
pg_dump --format=custom dbname > backup.dump
```

- Portable между версиями
- Медленнее restore (replay SQL)
- Можно восстановить отдельные таблицы

### Incremental / WAL-based

Базовый full backup + непрерывная архивация WAL.

```
Full backup (Sunday) + WAL segments (Mon-Sat)
  → Restore full → Replay WAL → PITR to any point
```

### Snapshot-based

Snapshot блочного устройства (EBS snapshot, Ceph snapshot).

- Очень быстрый (copy-on-write)
- Привязан к storage provider
- Может быть inconsistent без fsfreeze / checkpoint

## Point-in-Time Recovery (PITR)

Восстановление БД на произвольный момент времени.

```
Full backup          WAL archive
(00:00 Sun)    [Sun 00:00] ... [Tue 14:30] ... [Now]
    │                              │
    └── Restore full ──► Replay WAL до 14:30 Tue
```

**Реализация для PostgreSQL:**

```bash
# Continuous WAL archiving в S3
archive_command = 'wal-g wal-push %p'
archive_mode = on

# Restore
wal-g backup-fetch /var/lib/postgresql/data LATEST
# recovery.signal + recovery_target_time
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '2026-02-11 14:30:00+00'
```

## Backup Schedule

```yaml
backup:
  full:
    schedule: "0 3 * * 0"      # каждое воскресенье в 3:00
    retention: 4                 # хранить 4 full backup
  incremental:
    enabled: true                # WAL archiving
    retention_days: 7            # PITR окно — 7 дней
  storage:
    type: s3
    bucket: backups-prod
    encryption: AES-256
    region: eu-west-1
```

## Backup Storage

| Хранилище | Стоимость | Latency | Применение |
|-----------|-----------|---------|------------|
| Local disk | — | Минимальная | Не годится (тот же failure domain) |
| S3 / GCS | Низкая | Средняя | Стандарт для production |
| S3 Glacier / Archive | Очень низкая | Высокая (часы) | Long-term retention |
| Cross-region S3 | Средняя | Средняя | Disaster recovery |

## Restore Flow

```
User: POST /v1/instances/{id}/restore
{
  "target_time": "2026-02-11T14:30:00Z",
  "target_instance_name": "my-postgres-restored"
}

Flow:
1. Найти ближайший full backup до target_time
2. Создать новый инстанс
3. Restore full backup
4. Replay WAL до target_time
5. Открыть для подключений
6. Instance state: RUNNING
```

**Важно:** restore создаёт **новый инстанс**, не перезаписывает существующий. Это безопасно и позволяет проверить данные перед переключением.

## Инструменты

| Инструмент | СУБД | Описание |
|------------|------|----------|
| wal-g | PostgreSQL, MySQL, MongoDB | Облачный backup, сжатие, шифрование |
| pgBackRest | PostgreSQL | Full, incremental, parallel backup |
| Barman | PostgreSQL | Backup и recovery manager |
| Percona XtraBackup | MySQL | Hot physical backup |
| mongodump | MongoDB | Logical backup |

## Метрики бэкапов

- **RPO (Recovery Point Objective)** — максимально допустимая потеря данных. С WAL archiving → RPO ≈ 0 (потеря только незаархивированных WAL)
- **RTO (Recovery Time Objective)** — время восстановления. Зависит от размера backup + объёма WAL для replay
- **Backup success rate** — % успешных бэкапов
- **Backup duration** — время выполнения
- **Backup size** — для планирования storage

## Верификация бэкапов

Бэкап без проверки восстановления — не бэкап.

```
Периодически (еженедельно):
1. Restore последний backup в isolated environment
2. Запустить integrity checks
3. Выполнить test queries
4. Записать результат в audit log
```
