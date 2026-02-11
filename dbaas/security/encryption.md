# Encryption

Шифрование данных на всех уровнях: в transit, at rest, на уровне приложения.

## Encryption in Transit

TLS/SSL для всех соединений.

```
Client ──TLS 1.3──► Proxy ──TLS──► DB Engine
                                     │
                              Replication ──TLS──► Replica
```

### Конфигурация PostgreSQL

```
# postgresql.conf
ssl = on
ssl_cert_file = '/certs/server.crt'
ssl_key_file = '/certs/server.key'
ssl_ca_file = '/certs/ca.crt'
ssl_min_protocol_version = 'TLSv1.3'

# pg_hba.conf — требовать SSL
hostssl all all 0.0.0.0/0 scram-sha-256
```

### Клиентское подключение

```
postgresql://user:pass@host:5432/db?sslmode=verify-full&sslrootcert=ca.crt
```

| sslmode | Описание |
|---------|----------|
| `disable` | Без TLS |
| `require` | TLS, но без верификации сертификата |
| `verify-ca` | TLS + проверка CA |
| `verify-full` | TLS + проверка CA + hostname |

В DBaaS рекомендуется минимум `verify-ca`, идеально `verify-full`.

## Encryption at Rest

Шифрование данных на диске.

### Storage-level encryption

```
Data → Write → Encrypt (AES-256) → Disk
Disk → Read → Decrypt → Data
```

Прозрачно для СУБД. Реализуется на уровне storage:
- AWS EBS encryption (default KMS или CMK)
- GCP PD encryption (automatic)
- LUKS (Linux Unified Key Setup) для self-hosted
- Kubernetes: encrypted StorageClass

### Database-level encryption (TDE)

Transparent Data Encryption внутри СУБД:
- Шифрует tablespace/datafiles
- Поддерживается: Oracle, SQL Server, MySQL (Enterprise), PostgreSQL (через расширения)

### Envelope Encryption

```
Data → encrypt with DEK → encrypted data
DEK  → encrypt with KEK → encrypted DEK

Storage:
  [encrypted data] + [encrypted DEK]

KEK хранится в KMS (AWS KMS, GCP KMS, HashiCorp Vault)
```

- **DEK (Data Encryption Key)** — ключ для данных, генерируется для каждого инстанса
- **KEK (Key Encryption Key)** — мастер-ключ в KMS, шифрует DEK

### CMEK (Customer-Managed Encryption Keys)

Пользователь управляет KEK в своём KMS.

```
Customer KMS (their account):
  KEK: arn:aws:kms:eu-west-1:123456:key/abc-123

DBaaS Platform:
  DEK (encrypted by customer's KEK)
  Data (encrypted by DEK)
```

Пользователь может отозвать ключ → данные становятся недоступны (crypto shredding).

## Backup Encryption

```
Backup flow:
  pg_basebackup → compress (zstd) → encrypt (AES-256-GCM) → S3

Encryption key:
  Per-backup DEK → encrypted by KEK → stored alongside backup metadata
```

Бэкапы **всегда** должны быть зашифрованы. Даже если storage зашифрован — defense in depth.

## WAL Encryption

WAL содержит все данные → тоже нужно шифровать.

```
WAL archiving:
  wal-g wal-push --encryption-key=... %p → S3 (encrypted)
```

## Key Rotation

Периодическая смена ключей шифрования.

```
1. Generate new KEK in KMS
2. Re-encrypt all DEKs with new KEK
3. (Data re-encryption not needed — only DEKs change)
4. Retire old KEK after grace period

Schedule: every 90-365 days (compliance-dependent)
```

## Certificate Management

```
CA (Certificate Authority)
  └── Server cert (per instance, auto-rotated)
  └── Client cert (optional, for cert-based auth)

Rotation:
  1. Issue new cert (before expiry)
  2. Deploy to instance
  3. Reload (no restart needed for PostgreSQL)

Auto-rotation: cert-manager (Kubernetes) or internal PKI
```
