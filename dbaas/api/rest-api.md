# REST API Design

Дизайн REST API для DBaaS Control Plane.

## Структура URL

```
https://api.example.com/v1/projects/{project_id}/instances/{instance_id}
```

| Компонент | Описание |
|-----------|----------|
| `/v1/` | Версия API |
| `/projects/{id}/` | Скоупинг по проекту (tenant isolation) |
| `/instances/` | Ресурс |
| `/{instance_id}` | Конкретный инстанс |

## Основные endpoints

### Instances

```
POST   /v1/projects/{pid}/instances          — создать инстанс
GET    /v1/projects/{pid}/instances          — список инстансов
GET    /v1/projects/{pid}/instances/{id}     — получить инстанс
PATCH  /v1/projects/{pid}/instances/{id}     — обновить инстанс
DELETE /v1/projects/{pid}/instances/{id}     — удалить инстанс

POST   /v1/projects/{pid}/instances/{id}/restart    — рестарт
POST   /v1/projects/{pid}/instances/{id}/failover   — ручной failover
POST   /v1/projects/{pid}/instances/{id}/restore     — восстановить из бэкапа
```

### Backups

```
POST   /v1/projects/{pid}/instances/{id}/backups     — создать бэкап
GET    /v1/projects/{pid}/instances/{id}/backups     — список бэкапов
GET    /v1/projects/{pid}/backups/{bid}              — получить бэкап
DELETE /v1/projects/{pid}/backups/{bid}              — удалить бэкап
```

### Operations

```
GET    /v1/projects/{pid}/operations         — список операций
GET    /v1/projects/{pid}/operations/{oid}   — статус операции
POST   /v1/projects/{pid}/operations/{oid}/cancel — отменить операцию
```

### Users (DB users)

```
POST   /v1/projects/{pid}/instances/{id}/users       — создать пользователя
GET    /v1/projects/{pid}/instances/{id}/users       — список
PATCH  /v1/projects/{pid}/instances/{id}/users/{uid} — обновить
DELETE /v1/projects/{pid}/instances/{id}/users/{uid} — удалить
```

## Request / Response формат

### Create Instance

```http
POST /v1/projects/proj-456/instances
Content-Type: application/json
Authorization: Bearer <token>
Idempotency-Key: req-abc-123

{
  "name": "my-postgres",
  "engine": "postgresql",
  "version": "16",
  "region": "eu-west-1",
  "resources": {
    "cpu": 2,
    "memory_gb": 4,
    "storage_gb": 50
  }
}
```

### Response (202 Accepted)

```json
{
  "instance": {
    "id": "inst-abc123",
    "name": "my-postgres",
    "state": "CREATING",
    "engine": "postgresql",
    "version": "16",
    "region": "eu-west-1",
    "resources": {
      "cpu": 2,
      "memory_gb": 4,
      "storage_gb": 50
    },
    "endpoints": [],
    "created_at": "2026-02-11T10:00:00Z",
    "updated_at": "2026-02-11T10:00:00Z"
  },
  "operation": {
    "id": "op-xyz789",
    "type": "CREATE_INSTANCE",
    "state": "RUNNING",
    "progress_percent": 0,
    "created_at": "2026-02-11T10:00:00Z"
  }
}
```

## Pagination

```http
GET /v1/projects/proj-456/instances?page_size=20&page_token=eyJpZCI6MTAwfQ

Response:
{
  "instances": [...],
  "next_page_token": "eyJpZCI6MTIwfQ",
  "total_count": 47
}
```

Cursor-based pagination (не offset). Стабильна при добавлении/удалении элементов.

## Filtering & Sorting

```http
GET /v1/projects/proj-456/instances?filter=engine%3Dpostgresql&sort=-created_at
```

## Error Response

```json
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Instance limit exceeded: 10/10 instances used",
    "details": [
      {
        "resource": "instances",
        "limit": 10,
        "current": 10
      }
    ],
    "request_id": "req-abc-123"
  }
}
```

| HTTP Code | Когда |
|-----------|-------|
| 400 | Невалидный запрос |
| 401 | Не аутентифицирован |
| 403 | Нет прав |
| 404 | Ресурс не найден |
| 409 | Конфликт состояния (instance already deleting) |
| 422 | Валидация (engine not supported) |
| 429 | Rate limit |
| 500 | Internal error |

## Versioning

```
URL path: /v1/, /v2/
Header:   Accept: application/vnd.dbaas.v1+json

Правила:
- Breaking changes → новая major version
- Additive changes (новые поля) → обратно совместимо
- Deprecation: announce → sunset period (6 months) → remove
```

## Rate Limiting

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 600
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1707660000
```

Типичные лимиты: 600 req/min для read, 60 req/min для write.
