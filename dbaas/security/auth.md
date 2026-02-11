# Authentication & Authorization

Два уровня: доступ к платформе DBaaS (Control Plane) и доступ к самой БД (Data Plane).

## Control Plane Auth

### Authentication (кто ты?)

| Метод | Описание |
|-------|----------|
| API Key | Статический токен, простой, для автоматизации |
| OAuth 2.0 / OIDC | SSO через IdP (Google, GitHub, Okta) |
| Service Account | Для CI/CD и Terraform |
| mTLS | Взаимная аутентификация по сертификатам |

```
Client → API Gateway → Auth Service → verify token → API
                           │
                    ┌──────┴──────┐
                    │    IdP      │ (Okta, Auth0, Keycloak)
                    └─────────────┘
```

### Authorization (что можешь?)

RBAC — Role-Based Access Control.

```yaml
roles:
  admin:
    - instances.*        # полный доступ
    - backups.*
    - users.*
  developer:
    - instances.read
    - instances.connect  # подключение к БД
    - backups.read
    - logs.read
  viewer:
    - instances.read
    - metrics.read
```

Привязка: `User → Role → Project/Organization`

## Data Plane Auth (доступ к БД)

### Database Users

```sql
-- Создание через API платформы
POST /v1/instances/{id}/users
{
  "name": "app_user",
  "password": "auto-generated",
  "roles": ["readwrite"]
}

-- Или напрямую в БД
CREATE ROLE app_user WITH LOGIN PASSWORD '***';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

### Методы аутентификации (PostgreSQL)

| Метод | Описание |
|-------|----------|
| `scram-sha-256` | Стандарт, безопасный (рекомендуется) |
| `md5` | Устаревший, слабый |
| `cert` | Аутентификация по клиентскому сертификату |
| `ldap` | Аутентификация через LDAP/AD |

```
# pg_hba.conf (управляется платформой)
hostssl all app_user  10.0.0.0/8  scram-sha-256
hostssl all admin     0.0.0.0/0   cert
```

### IAM Authentication

Подключение к БД через облачный IAM (без паролей).

```
AWS RDS IAM auth:
  1. IAM policy разрешает rds-db:connect
  2. Приложение получает temporary token через AWS SDK
  3. Использует token как пароль при подключении

GCP Cloud SQL:
  1. Cloud SQL Auth Proxy → IAM → connection
  2. Или direct IAM database authentication
```

## Принцип наименьших привилегий

```sql
-- Read-only для аналитики
CREATE ROLE analyst WITH LOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO analyst;

-- Application user: CRUD без DDL
CREATE ROLE app WITH LOGIN;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app;
REVOKE CREATE ON SCHEMA public FROM app;

-- Migration user: DDL
CREATE ROLE migrator WITH LOGIN;
GRANT ALL ON SCHEMA public TO migrator;
```

## Audit logging

```sql
-- Включение pgAudit
CREATE EXTENSION pgaudit;
SET pgaudit.log = 'all';

-- Логируется:
-- [pgaudit] user=admin action=CREATE TABLE table=users
-- [pgaudit] user=app action=SELECT table=orders rows=150
```
