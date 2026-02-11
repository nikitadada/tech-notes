# Idempotency

Повторный запрос с теми же параметрами даёт тот же результат. Критично для API, где сетевые ошибки — норма.

## Проблема

```
Client → POST /instances → 200 OK (instance created)
         ↕ network timeout — клиент не получил ответ
Client → POST /instances → ??? создать ещё один инстанс?
```

Без idempotency: дублирование ресурсов, двойное списание.

## Idempotency Key

Клиент передаёт уникальный ключ. Сервер гарантирует: один ключ = одна операция.

```http
POST /v1/projects/proj-456/instances
Idempotency-Key: req-abc-123-def-456
Content-Type: application/json

{
  "name": "my-postgres",
  "engine": "postgresql"
}
```

### Поведение

| Ситуация | Результат |
|----------|-----------|
| Первый запрос | Создаёт инстанс, сохраняет ответ |
| Повторный запрос (тот же ключ) | Возвращает сохранённый ответ |
| Тот же ключ, другое тело | 422 Unprocessable (conflict) |
| Запрос без ключа | Каждый вызов создаёт новый ресурс |

## Реализация

### Storage

```sql
CREATE TABLE idempotency_keys (
    key         TEXT PRIMARY KEY,
    project_id  TEXT NOT NULL,
    method      TEXT NOT NULL,
    path        TEXT NOT NULL,
    request_hash TEXT NOT NULL,    -- hash тела запроса
    status_code INT,
    response    JSONB,
    created_at  TIMESTAMPTZ DEFAULT now(),
    expires_at  TIMESTAMPTZ DEFAULT now() + interval '24 hours'
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
```

### Server-side logic

```go
func (h *Handler) CreateInstance(w http.ResponseWriter, r *http.Request) {
    idempotencyKey := r.Header.Get("Idempotency-Key")

    if idempotencyKey != "" {
        // 1. Проверить существующий ключ
        existing, err := h.store.GetIdempotencyKey(r.Context(), idempotencyKey)
        if err == nil {
            // Проверить что тело запроса совпадает
            if existing.RequestHash != hashRequest(r) {
                http.Error(w, "idempotency key reused with different request", 422)
                return
            }
            // Вернуть сохранённый ответ
            w.WriteHeader(existing.StatusCode)
            w.Write(existing.Response)
            return
        }

        // 2. Захватить ключ (с блокировкой для предотвращения race condition)
        locked, err := h.store.LockIdempotencyKey(r.Context(), idempotencyKey, hashRequest(r))
        if err != nil {
            // Другой запрос уже обрабатывает этот ключ
            http.Error(w, "request in progress", 409)
            return
        }
        defer locked.Release()
    }

    // 3. Выполнить операцию
    instance, op, err := h.service.CreateInstance(r.Context(), req)

    // 4. Сохранить результат для idempotency key
    if idempotencyKey != "" {
        h.store.SaveIdempotencyResult(r.Context(), idempotencyKey, statusCode, response)
    }

    // 5. Вернуть ответ
    writeJSON(w, http.StatusAccepted, response)
}
```

### Race condition protection

Два одинаковых запроса одновременно:

```go
// Атомарный INSERT с advisory lock
func (s *Store) LockIdempotencyKey(ctx context.Context, key, requestHash string) (*Lock, error) {
    // pg_advisory_xact_lock гарантирует что только один запрос проходит
    _, err := s.db.Exec(ctx, `SELECT pg_advisory_xact_lock(hashtext($1))`, key)
    if err != nil {
        return nil, err
    }

    // INSERT ON CONFLICT — atomic check-and-insert
    _, err = s.db.Exec(ctx, `
        INSERT INTO idempotency_keys (key, project_id, request_hash, status_code)
        VALUES ($1, $2, $3, NULL)
        ON CONFLICT (key) DO NOTHING
    `, key, projectID, requestHash)

    return &Lock{key: key}, err
}
```

## Естественная идемпотентность

Некоторые операции идемпотентны по природе:

| Метод | Идемпотентный? | Нужен ключ? |
|-------|:-:|:-:|
| GET | Да | Нет |
| PUT | Да (replace) | Нет |
| DELETE | Да | Нет |
| PATCH | Зависит | Рекомендуется |
| POST | Нет | Да |

## Client-side

```go
func createInstanceWithRetry(ctx context.Context, client *Client, req CreateReq) (*Instance, error) {
    idempotencyKey := uuid.New().String()

    var lastErr error
    for attempt := 0; attempt < 3; attempt++ {
        instance, err := client.CreateInstance(ctx, req, idempotencyKey)
        if err == nil {
            return instance, nil
        }

        lastErr = err
        if !isRetryable(err) {
            return nil, err
        }

        time.Sleep(backoff(attempt))
        // Тот же idempotencyKey → безопасный retry
    }

    return nil, fmt.Errorf("failed after 3 attempts: %w", lastErr)
}
```

## Expiration

Idempotency ключи имеют TTL (обычно 24 часа). После TTL — ключ удаляется, повторный запрос создаст новый ресурс.

```sql
-- Cleanup expired keys (cron job)
DELETE FROM idempotency_keys WHERE expires_at < now();
```

## Best practices

1. **Клиент генерирует ключ** (UUID v4) — не сервер
2. **Один ключ = один intent** — новый ресурс = новый ключ
3. **Храни ключи ≥24 часов** — покрыть retry window
4. **Верифицируй тело запроса** — тот же ключ с другим телом = ошибка
5. **Документируй** — какие endpoints требуют idempotency key
