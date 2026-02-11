# Stacks & Queues

## Stack (LIFO)

Последний пришёл — первый вышел. Все операции O(1).

```go
// Stack через slice
type Stack[T any] struct {
    data []T
}

func (s *Stack[T]) Push(val T)    { s.data = append(s.data, val) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.data) == 0 {
        var zero T
        return zero, false
    }
    val := s.data[len(s.data)-1]
    s.data = s.data[:len(s.data)-1]
    return val, true
}
func (s *Stack[T]) Peek() (T, bool) {
    if len(s.data) == 0 {
        var zero T
        return zero, false
    }
    return s.data[len(s.data)-1], true
}
func (s *Stack[T]) Len() int { return len(s.data) }
```

### Применение стека

- **Скобочные последовательности** — валидация вложенности
- **Обратная польская нотация** — вычисление выражений
- **DFS** — итеративная реализация
- **Undo/Redo** — история операций
- **Call stack** — стек вызовов функций

### Monotonic Stack

Стек, в котором элементы всегда отсортированы (по возрастанию или убыванию).

```go
// Next Greater Element: для каждого элемента найти ближайший бо́льший справа
func nextGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }
    stack := []int{} // индексы

    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[i] > nums[stack[len(stack)-1]] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }
    return result
}
// [2,1,4,3] → [4,4,-1,-1]
```

**Применение:** next greater/smaller element, largest rectangle in histogram, trapping rain water.

## Queue (FIFO)

Первый пришёл — первый вышел. Все операции O(1).

```go
// Queue через slice (простая реализация)
type Queue[T any] struct {
    data []T
}

func (q *Queue[T]) Enqueue(val T)    { q.data = append(q.data, val) }
func (q *Queue[T]) Dequeue() (T, bool) {
    if len(q.data) == 0 {
        var zero T
        return zero, false
    }
    val := q.data[0]
    q.data = q.data[1:]
    return val, true
}
func (q *Queue[T]) Len() int { return len(q.data) }
```

**Замечание:** `q.data[1:]` не освобождает память. Для production — ring buffer или `container/list`.

### Ring Buffer (Circular Queue)

```go
type RingBuffer[T any] struct {
    data       []T
    head, tail int
    size, cap  int
}

func NewRingBuffer[T any](capacity int) *RingBuffer[T] {
    return &RingBuffer[T]{
        data: make([]T, capacity),
        cap:  capacity,
    }
}

func (rb *RingBuffer[T]) Enqueue(val T) bool {
    if rb.size == rb.cap {
        return false // полон
    }
    rb.data[rb.tail] = val
    rb.tail = (rb.tail + 1) % rb.cap
    rb.size++
    return true
}

func (rb *RingBuffer[T]) Dequeue() (T, bool) {
    if rb.size == 0 {
        var zero T
        return zero, false
    }
    val := rb.data[rb.head]
    rb.head = (rb.head + 1) % rb.cap
    rb.size--
    return val, true
}
```

### Применение очереди

- **BFS** — обход графа в ширину
- **Task queue** — очередь задач для worker pool
- **Rate limiting** — sliding window
- **Буферизация** — потоки данных

## Deque (Double-Ended Queue)

Вставка и удаление с обоих концов за O(1).

```go
import "container/list"

deque := list.New()
deque.PushFront(1)
deque.PushBack(2)
front := deque.Remove(deque.Front()) // 1
back := deque.Remove(deque.Back())   // 2
```

### Sliding Window Maximum (deque)

```go
func maxSlidingWindow(nums []int, k int) []int {
    var result []int
    deque := []int{} // индексы, значения по убыванию

    for i, n := range nums {
        // Убрать элементы за окном
        for len(deque) > 0 && deque[0] <= i-k {
            deque = deque[1:]
        }
        // Убрать меньшие элементы с конца
        for len(deque) > 0 && nums[deque[len(deque)-1]] <= n {
            deque = deque[:len(deque)-1]
        }
        deque = append(deque, i)
        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    return result
}
```
