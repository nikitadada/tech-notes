# Channels

Типизированный канал для коммуникации между горутинами. Основной примитив конкурентности в Go.

## Типы каналов

```go
ch := make(chan int)      // unbuffered — блокирует отправителя до чтения
ch := make(chan int, 10)  // buffered — блокирует только при заполнении буфера
```

### Unbuffered

Синхронизация: отправитель блокируется, пока получатель не прочитает.

```go
ch := make(chan int)

go func() {
    ch <- 42  // блокируется до чтения
}()

val := <-ch  // 42
```

### Buffered

Отправитель блокируется только когда буфер полон.

```go
ch := make(chan int, 3)
ch <- 1  // не блокируется
ch <- 2  // не блокируется
ch <- 3  // не блокируется
ch <- 4  // блокируется — буфер полон
```

## Направление каналов

```go
func producer(out chan<- int) { // только отправка
    out <- 42
}

func consumer(in <-chan int) {  // только чтение
    val := <-in
}
```

Направление в сигнатуре функции — compile-time защита от неправильного использования.

## Закрытие каналов

```go
close(ch)

// Проверка закрыт ли канал
val, ok := <-ch  // ok == false если канал закрыт и пуст

// Итерация до закрытия
for val := range ch {
    process(val)
}
```

**Правила:**
- Закрывает только отправитель
- Отправка в закрытый канал — panic
- Чтение из закрытого канала возвращает zero value

## Select

Мультиплексирование операций с каналами.

```go
select {
case msg := <-msgCh:
    handle(msg)
case err := <-errCh:
    log.Error(err)
case <-time.After(5 * time.Second):
    log.Warn("timeout")
case <-ctx.Done():
    return ctx.Err()
default:
    // не блокируется если все каналы не готовы
}
```

- Если несколько case готовы — выбирается случайно
- `default` делает select неблокирующим

## Паттерны

### Fan-out / Fan-in

```go
// Fan-out: несколько горутин читают из одного канала
func fanOut(in <-chan int, workers int) []<-chan int {
    outs := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        outs[i] = worker(in)
    }
    return outs
}

// Fan-in: объединение нескольких каналов в один
func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)

    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                merged <- v
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

### Done channel (отмена)

```go
done := make(chan struct{})

go func() {
    defer close(done)
    // работа
}()

select {
case <-done:
    fmt.Println("finished")
case <-time.After(timeout):
    fmt.Println("timeout")
}
```

### Semaphore

```go
sem := make(chan struct{}, maxConcurrency)

for _, task := range tasks {
    sem <- struct{}{} // захватить слот
    go func(t Task) {
        defer func() { <-sem }() // освободить слот
        process(t)
    }(task)
}
```
