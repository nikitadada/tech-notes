# Go Exercises

Практические задачи для тренировки конкурентных паттернов.

## 1. Rate Limiter

Реализуй rate limiter, ограничивающий N запросов в секунду.

**Требования:**
- `func NewRateLimiter(rps int) *RateLimiter`
- `func (rl *RateLimiter) Allow() bool` — разрешить или отклонить запрос
- Должен работать корректно из нескольких горутин

**Подсказка:** token bucket алгоритм с `time.Ticker`.

---

## 2. Parallel Map

Реализуй функцию, которая применяет трансформацию к каждому элементу слайса параллельно с ограничением concurrency.

```go
func ParallelMap[T, R any](ctx context.Context, items []T, maxWorkers int, fn func(context.Context, T) (R, error)) ([]R, error)
```

**Требования:**
- Сохранение порядка результатов
- Отмена через context
- Возврат первой ошибки

---

## 3. Bounded Concurrent Cache

Реализуй потокобезопасный кэш с ограничением по количеству элементов.

```go
type Cache[K comparable, V any] struct { ... }

func NewCache[K comparable, V any](maxSize int) *Cache[K, V]
func (c *Cache[K, V]) Get(key K) (V, bool)
func (c *Cache[K, V]) Set(key K, value V)
```

**Требования:**
- LRU eviction при превышении maxSize
- Потокобезопасность
- O(1) Get и Set

---

## 4. Pipeline Builder

Построй configurable pipeline для обработки строк.

```go
pipeline := NewPipeline[string]().
    AddStage(trim).
    AddStage(toLower).
    AddStage(validate).
    WithWorkers(4).
    Build()

results, err := pipeline.Process(ctx, input)
```

**Требования:**
- Произвольное количество стадий
- Параллелизм на каждой стадии
- Graceful cancellation

---

## 5. Pub/Sub Broker

Реализуй in-memory pub/sub брокер.

```go
type Broker[T any] struct { ... }

func (b *Broker[T]) Subscribe(topic string) <-chan T
func (b *Broker[T]) Unsubscribe(topic string, ch <-chan T)
func (b *Broker[T]) Publish(topic string, msg T)
func (b *Broker[T]) Close()
```

**Требования:**
- Несколько подписчиков на один топик
- Неблокирующий Publish (медленный подписчик не тормозит остальных)
- Корректная очистка при Close/Unsubscribe

---

## Шаблон для решений

```go
package main

import (
    "context"
    "fmt"
    "testing"
)

// Решение здесь

func TestSolution(t *testing.T) {
    // Тесты здесь
}
```
