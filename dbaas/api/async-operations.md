# Async Operations

Большинство операций в DBaaS (создание, масштабирование, бэкап) занимают от секунд до минут. Реализуются как асинхронные операции.

## Паттерн

```
Client → POST /instances → 202 Accepted + Operation ID
Client → GET /operations/{id} → poll status
Client → GET /operations/{id} → poll status
Client → GET /operations/{id} → COMPLETED ✓
```

## Operation Resource

```json
{
  "id": "op-xyz789",
  "type": "CREATE_INSTANCE",
  "state": "RUNNING",
  "progress_percent": 65,
  "resource": {
    "type": "instance",
    "id": "inst-abc123"
  },
  "steps": [
    {"name": "validate", "state": "COMPLETED", "duration_ms": 50},
    {"name": "allocate_resources", "state": "COMPLETED", "duration_ms": 45000},
    {"name": "deploy_database", "state": "RUNNING", "started_at": "..."},
    {"name": "setup_replication", "state": "PENDING"},
    {"name": "register_endpoints", "state": "PENDING"}
  ],
  "created_at": "2026-02-11T10:00:00Z",
  "updated_at": "2026-02-11T10:01:05Z",
  "completed_at": null,
  "error": null
}
```

## Состояния операции

```
PENDING → RUNNING → COMPLETED
                  → FAILED
                  → CANCELLED
```

| State | Описание |
|-------|----------|
| `PENDING` | В очереди, ещё не начата |
| `RUNNING` | Выполняется |
| `COMPLETED` | Успешно завершена |
| `FAILED` | Завершена с ошибкой |
| `CANCELLED` | Отменена пользователем |

## Polling

### Simple polling

```go
func waitForOperation(ctx context.Context, client *Client, opID string) (*Operation, error) {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-ticker.C:
            op, err := client.GetOperation(ctx, opID)
            if err != nil {
                return nil, err
            }
            if op.State == "COMPLETED" || op.State == "FAILED" || op.State == "CANCELLED" {
                return op, nil
            }
        }
    }
}
```

### Exponential backoff polling

```go
func waitWithBackoff(ctx context.Context, client *Client, opID string) (*Operation, error) {
    interval := 2 * time.Second
    maxInterval := 30 * time.Second

    for {
        op, err := client.GetOperation(ctx, opID)
        if err != nil {
            return nil, err
        }
        if op.IsTerminal() {
            return op, nil
        }

        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(interval):
            interval = min(interval*2, maxInterval)
        }
    }
}
```

### Long-polling

```http
GET /v1/operations/op-xyz789?wait=30s

# Сервер держит соединение до 30 секунд
# Отвечает сразу при изменении состояния
# Или по таймауту с текущим состоянием
```

### Webhooks

```json
POST /v1/projects/{pid}/webhooks
{
  "url": "https://my-app.com/hooks/dbaas",
  "events": ["operation.completed", "operation.failed"],
  "secret": "webhook-signing-secret"
}
```

При завершении операции:

```json
POST https://my-app.com/hooks/dbaas
X-Webhook-Signature: sha256=...

{
  "event": "operation.completed",
  "operation": {
    "id": "op-xyz789",
    "type": "CREATE_INSTANCE",
    "state": "COMPLETED",
    "resource": {"type": "instance", "id": "inst-abc123"}
  },
  "timestamp": "2026-02-11T10:02:00Z"
}
```

## Отмена операции

```http
POST /v1/operations/op-xyz789/cancel

Response: 200 OK
{
  "state": "CANCELLING"
}
```

Не все операции отменяемы. Некоторые шаги необратимы.

```go
func (o *Operation) Cancel(ctx context.Context) error {
    if !o.IsCancellable() {
        return ErrOperationNotCancellable
    }

    // Отменить текущий шаг
    if err := o.currentStep.Cancel(ctx); err != nil {
        return err
    }

    // Rollback завершённых шагов (в обратном порядке)
    for i := len(o.completedSteps) - 1; i >= 0; i-- {
        if err := o.completedSteps[i].Rollback(ctx); err != nil {
            o.State = "FAILED"
            return err
        }
    }

    o.State = "CANCELLED"
    return nil
}
```

## Retention

Операции хранятся в metadata DB.

```
Active operations: в основной таблице
Completed (< 30 days): в основной таблице, доступны через API
Completed (> 30 days): архивируются, недоступны через API
```

## List Operations

```http
GET /v1/projects/{pid}/operations?resource_id=inst-abc123&state=RUNNING

{
  "operations": [
    {
      "id": "op-xyz789",
      "type": "MODIFY_INSTANCE",
      "state": "RUNNING",
      "progress_percent": 45,
      "created_at": "2026-02-11T10:00:00Z"
    }
  ]
}
```

Полезно для UI: показать текущие операции над инстансом.
