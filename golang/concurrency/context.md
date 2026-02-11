# Context

`context.Context` — механизм для передачи дедлайнов, сигналов отмены и request-scoped значений через цепочку вызовов.

## Создание контекста

```go
// Базовые (корневые)
ctx := context.Background()  // для main, init, тестов
ctx := context.TODO()        // плейсхолдер, когда неясно какой ctx использовать

// С отменой
ctx, cancel := context.WithCancel(parentCtx)
defer cancel()  // ВСЕГДА вызывай cancel чтобы освободить ресурсы

// С таймаутом
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

// С дедлайном
ctx, cancel := context.WithDeadline(parentCtx, time.Now().Add(30*time.Second))
defer cancel()

// Со значениями (используй осторожно)
ctx = context.WithValue(parentCtx, key, value)
```

## Проверка отмены

```go
select {
case <-ctx.Done():
    return ctx.Err()  // context.Canceled или context.DeadlineExceeded
default:
    // продолжаем работу
}
```

## Пример: HTTP handler с таймаутом

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    result, err := fetchData(ctx)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(result)
}
```

## Пример: передача контекста в запрос

```go
func fetchData(ctx context.Context) (*Data, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err  // вернёт ошибку если ctx отменён
    }
    defer resp.Body.Close()

    var data Data
    return &data, json.NewDecoder(resp.Body).Decode(&data)
}
```

## Пример: горутина с контекстом

```go
func worker(ctx context.Context, jobs <-chan Job) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        case job, ok := <-jobs:
            if !ok {
                return nil
            }
            if err := process(ctx, job); err != nil {
                return err
            }
        }
    }
}
```

## context.WithValue — передача значений

```go
type ctxKey string

const requestIDKey ctxKey = "request_id"

// Установка
ctx = context.WithValue(ctx, requestIDKey, "abc-123")

// Чтение
if reqID, ok := ctx.Value(requestIDKey).(string); ok {
    log.Printf("request_id=%s", reqID)
}
```

**Правила для WithValue:**
- Используй для request-scoped данных (request ID, auth token)
- Не используй для передачи опциональных параметров — используй аргументы
- Ключ должен быть неэкспортируемого типа чтобы избежать коллизий

## Best practices

1. `context.Context` — всегда первый параметр функции: `func Do(ctx context.Context, ...)`
2. Не храни ctx в структурах — передавай явно
3. Всегда вызывай `cancel()` (обычно через `defer`)
4. Проверяй `ctx.Done()` в долгих операциях и циклах
5. Используй `context.WithValue` минимально — только для cross-cutting concerns
