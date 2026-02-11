# Goroutines

Легковесные потоки, управляемые Go runtime. Стоимость создания ~2KB стека (растёт динамически). Можно запускать тысячи горутин без проблем.

## Основы

```go
func main() {
    go doWork()       // запуск горутины
    go func() {       // анонимная горутина
        fmt.Println("async")
    }()
    time.Sleep(time.Second) // плохой способ ожидания
}
```

## WaitGroup — ожидание завершения

```go
func main() {
    var wg sync.WaitGroup

    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            fmt.Printf("worker %d\n", id)
        }(i)
    }

    wg.Wait() // блокируется до завершения всех горутин
}
```

**Важно:** передавай переменную цикла как аргумент горутины, иначе возможен data race (в Go < 1.22).

## errgroup — WaitGroup с ошибками

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) error {
    g, ctx := errgroup.WithContext(ctx)

    for _, url := range urls {
        g.Go(func() error {
            return fetch(ctx, url)
        })
    }

    return g.Wait() // возвращает первую ошибку
}
```

`errgroup` при ошибке в любой горутине отменяет ctx для остальных.

## Mutex — защита общих данных

```go
type SafeCounter struct {
    mu sync.Mutex
    v  map[string]int
}

func (c *SafeCounter) Inc(key string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.v[key]++
}
```

- `sync.Mutex` — эксклюзивная блокировка
- `sync.RWMutex` — множественные читатели, один писатель

## sync.Once — однократное выполнение

```go
var (
    instance *DB
    once     sync.Once
)

func GetDB() *DB {
    once.Do(func() {
        instance = connectDB()
    })
    return instance
}
```

## Детектор гонок

```bash
go run -race main.go
go test -race ./...
```

Всегда запускай тесты с `-race` в CI.

## Правила

1. Не запускай горутину, если не знаешь, когда она завершится
2. Передавай context для управления временем жизни
3. Используй `-race` для обнаружения data race
4. Закрывай каналы на стороне отправителя
5. Предпочитай каналы мьютексам для координации (share memory by communicating)
