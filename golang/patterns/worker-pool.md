# Worker Pool

Фиксированное количество горутин обрабатывает задачи из общей очереди. Контролирует параллелизм и потребление ресурсов.

## Базовая реализация

```go
func workerPool(ctx context.Context, jobs <-chan Job, results chan<- Result, numWorkers int) {
    var wg sync.WaitGroup

    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    result := process(job)
                    select {
                    case results <- result:
                    case <-ctx.Done():
                        return
                    }
                }
            }
        }(i)
    }

    wg.Wait()
    close(results)
}
```

## С errgroup

```go
func processAll(ctx context.Context, items []Item) ([]Result, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10) // максимум 10 горутин

    results := make([]Result, len(items))

    for i, item := range items {
        g.Go(func() error {
            res, err := process(ctx, item)
            if err != nil {
                return err
            }
            results[i] = res // безопасно — каждая горутина пишет в свой индекс
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

## Generics-версия (Go 1.18+)

```go
func Pool[T any, R any](ctx context.Context, workers int, input []T, fn func(context.Context, T) (R, error)) ([]R, error) {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(workers)

    results := make([]R, len(input))

    for i, item := range input {
        g.Go(func() error {
            res, err := fn(ctx, item)
            if err != nil {
                return err
            }
            results[i] = res
            return nil
        })
    }

    return results, g.Wait()
}

// Использование
users, err := Pool(ctx, 5, userIDs, fetchUser)
```

## Динамический worker pool

Пул, который масштабирует количество воркеров в зависимости от нагрузки.

```go
type DynamicPool struct {
    jobs    chan Job
    results chan Result
    minW    int
    maxW    int
}

func NewDynamicPool(minWorkers, maxWorkers, queueSize int) *DynamicPool {
    return &DynamicPool{
        jobs:    make(chan Job, queueSize),
        results: make(chan Result, queueSize),
        minW:    minWorkers,
        maxW:    maxWorkers,
    }
}

func (p *DynamicPool) Run(ctx context.Context) {
    // Запускаем минимум воркеров
    for i := 0; i < p.minW; i++ {
        go p.worker(ctx)
    }

    // Мониторинг очереди — добавляем воркеров при нагрузке
    go func() {
        currentWorkers := p.minW
        ticker := time.NewTicker(100 * time.Millisecond)
        defer ticker.Stop()

        for {
            select {
            case <-ctx.Done():
                return
            case <-ticker.C:
                pending := len(p.jobs)
                if pending > 0 && currentWorkers < p.maxW {
                    go p.worker(ctx)
                    currentWorkers++
                }
            }
        }
    }()
}
```

## Когда использовать

- HTTP-клиент с ограничением concurrency
- Пакетная обработка данных (файлы, записи БД)
- Rate-limited API вызовы
- Параллельная обработка сообщений из очереди
