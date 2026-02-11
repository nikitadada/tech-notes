# Pipelines

Цепочка стадий обработки данных, соединённых каналами. Каждая стадия: получает данные → обрабатывает → отправляет дальше.

## Базовый pipeline

```go
// Стадия 1: генерация
func generate(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case out <- n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Стадия 2: обработка (возведение в квадрат)
func square(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}

// Стадия 3: фильтрация
func filter(ctx context.Context, in <-chan int, predicate func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if predicate(n) {
                select {
                case out <- n:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}

// Использование
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    nums := generate(ctx, 1, 2, 3, 4, 5)
    squared := square(ctx, nums)
    big := filter(ctx, squared, func(n int) bool { return n > 10 })

    for val := range big {
        fmt.Println(val) // 16, 25
    }
}
```

## Fan-out / Fan-in pipeline

Распараллеливание тяжёлой стадии.

```go
func heavyStage(ctx context.Context, in <-chan Data) <-chan Result {
    // Fan-out: запуск нескольких воркеров
    numWorkers := runtime.NumCPU()
    channels := make([]<-chan Result, numWorkers)
    for i := 0; i < numWorkers; i++ {
        channels[i] = worker(ctx, in)
    }

    // Fan-in: объединение результатов
    return merge(ctx, channels...)
}

func merge[T any](ctx context.Context, channels ...<-chan T) <-chan T {
    var wg sync.WaitGroup
    out := make(chan T)

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan T) {
            defer wg.Done()
            for val := range c {
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

## Generic pipeline builder

```go
type Stage[In, Out any] func(context.Context, <-chan In) <-chan Out

func Pipe[A, B, C any](
    ctx context.Context,
    source <-chan A,
    stage1 Stage[A, B],
    stage2 Stage[B, C],
) <-chan C {
    return stage2(ctx, stage1(ctx, source))
}
```

## Батчинг в pipeline

Группировка элементов для пакетной обработки (например, bulk insert в БД).

```go
func batch[T any](ctx context.Context, in <-chan T, size int, timeout time.Duration) <-chan []T {
    out := make(chan []T)
    go func() {
        defer close(out)
        buf := make([]T, 0, size)
        timer := time.NewTimer(timeout)
        defer timer.Stop()

        for {
            select {
            case val, ok := <-in:
                if !ok {
                    if len(buf) > 0 {
                        out <- buf
                    }
                    return
                }
                buf = append(buf, val)
                if len(buf) >= size {
                    out <- buf
                    buf = make([]T, 0, size)
                    timer.Reset(timeout)
                }
            case <-timer.C:
                if len(buf) > 0 {
                    out <- buf
                    buf = make([]T, 0, size)
                }
                timer.Reset(timeout)
            case <-ctx.Done():
                return
            }
        }
    }()
    return out
}
```

## Правила

1. Каждая стадия закрывает свой выходной канал (`defer close(out)`)
2. Всегда проверяй `ctx.Done()` в select — иначе горутина утечёт
3. Upstream закрывает канал → downstream получает сигнал через `range`
4. Для отмены всего pipeline — `cancel()` корневого контекста
