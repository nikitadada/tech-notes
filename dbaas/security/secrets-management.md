# Secrets Management

Управление чувствительными данными: пароли БД, ключи шифрования, сертификаты.

## Типы секретов в DBaaS

| Секрет | Где используется |
|--------|-----------------|
| DB user passwords | Подключение приложения к БД |
| Replication credentials | Аутентификация между primary и replicas |
| TLS certificates | Шифрование соединений |
| Encryption keys (DEK) | Encryption at rest |
| API keys | Доступ к Control Plane |
| Backup encryption keys | Шифрование бэкапов |
| Agent tokens | Аутентификация agent ↔ CP |

## Хранение секретов

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: inst-abc123-credentials
  namespace: tenant-abc
type: Opaque
data:
  password: <base64>
  replication-password: <base64>
```

**Ограничения:**
- Base64 — не шифрование
- Доступны всем с RBAC на namespace
- Хранятся в etcd (нужно encryption at rest для etcd)

### External Secret Stores

| Хранилище | Описание |
|-----------|----------|
| HashiCorp Vault | Self-hosted, KV + dynamic secrets + PKI |
| AWS Secrets Manager | Managed, rotation, IAM integration |
| GCP Secret Manager | Managed, versioned, IAM |
| Azure Key Vault | Managed, HSM-backed |

### External Secrets Operator (Kubernetes)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault
    kind: ClusterSecretStore
  target:
    name: inst-abc123-credentials
  data:
    - secretKey: password
      remoteRef:
        key: dbaas/instances/abc123
        property: password
```

Автоматически синхронизирует секреты из Vault → Kubernetes Secret.

## Генерация паролей

```go
func generatePassword(length int) (string, error) {
    const charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%"
    b := make([]byte, length)
    for i := range b {
        n, err := rand.Int(rand.Reader, big.NewInt(int64(len(charset))))
        if err != nil {
            return "", err
        }
        b[i] = charset[n.Int64()]
    }
    return string(b), nil
}
```

Минимальные требования: 24+ символов, crypto/rand, без словарных слов.

## Rotation

Автоматическая смена секретов по расписанию.

### Password rotation

```
1. Generate new password
2. Create new DB user or ALTER existing
3. Update secret store
4. Verify new credentials work
5. Revoke old password (after grace period)

Schedule: every 90 days (or per compliance policy)
```

### Certificate rotation

```
1. Issue new certificate (before expiry, e.g., 30 days before)
2. Deploy to instance
3. Reload DB (pg_reload_conf — no restart)
4. Verify TLS with new cert
5. Revoke old cert

Auto: cert-manager handles this in Kubernetes
```

### Dynamic Secrets (Vault)

Vault генерирует temporary credentials с TTL.

```
App → Vault: "give me DB creds"
Vault → DB: CREATE ROLE temp_user_123 WITH LOGIN PASSWORD '***' VALID UNTIL '...'
Vault → App: {user: temp_user_123, password: '***', ttl: 1h}

After TTL:
Vault → DB: DROP ROLE temp_user_123
```

- Нет долгоживущих паролей
- Автоматическая ревокация
- Audit trail в Vault

## Доставка секретов приложению

| Метод | Описание |
|-------|----------|
| Environment variables | Простой, но видны в process list |
| Mounted files | Secret как файл в контейнере |
| Vault Agent sidecar | Автоматический fetch и renewal |
| SDK / API call | Приложение запрашивает у Vault напрямую |
| Connection string from API | DBaaS API возвращает connection string |

## Принципы

1. **Никогда не хардкоди секреты** в коде или конфигах
2. **Шифруй at rest** — etcd encryption, KMS-backed stores
3. **Минимальный TTL** — предпочитай dynamic secrets
4. **Audit** — логируй все доступы к секретам
5. **Rotation** — автоматическая ротация по расписанию
6. **Least privilege** — приложение получает только нужные секреты
