# Heaps (Кучи)

Дерево с heap-свойством: родитель всегда ≤ (min-heap) или ≥ (max-heap) детей.

## Свойства

| Операция | Сложность |
|----------|-----------|
| Получить min/max | O(1) |
| Вставка | O(log n) |
| Удаление min/max | O(log n) |
| Построение из массива | O(n) |
| Поиск произвольного | O(n) |

## Реализация через массив

```
         1
        / \
       3   2
      / \
     7   4

Массив: [1, 3, 2, 7, 4]
Индексы: parent(i) = (i-1)/2
          left(i)  = 2*i + 1
          right(i) = 2*i + 2
```

## Heap в Go

Go использует `container/heap` с интерфейсом:

```go
type IntHeap []int

func (h IntHeap) Len() int           { return len(h) }
func (h IntHeap) Less(i, j int) bool { return h[i] < h[j] } // min-heap
func (h IntHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *IntHeap) Push(x any) {
    *h = append(*h, x.(int))
}

func (h *IntHeap) Pop() any {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

// Использование
h := &IntHeap{5, 3, 8, 1}
heap.Init(h)
heap.Push(h, 2)
min := heap.Pop(h).(int) // 1
```

## Priority Queue

```go
type Item struct {
    Value    string
    Priority int
    Index    int
}

type PQ []*Item

func (pq PQ) Len() int            { return len(pq) }
func (pq PQ) Less(i, j int) bool  { return pq[i].Priority < pq[j].Priority }
func (pq PQ) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
    pq[i].Index = i
    pq[j].Index = j
}

func (pq *PQ) Push(x any)  { /* ... */ }
func (pq *PQ) Pop() any    { /* ... */ }

// Обновление приоритета
func (pq *PQ) Update(item *Item, priority int) {
    item.Priority = priority
    heap.Fix(pq, item.Index)
}
```

## Типичные задачи

### Top K элементов

Поддерживай min-heap размера K. Для top K наибольших — если новый элемент > min heap → заменяй.

```go
func topK(nums []int, k int) []int {
    h := &IntHeap{}
    heap.Init(h)
    for _, n := range nums {
        heap.Push(h, n)
        if h.Len() > k {
            heap.Pop(h)
        }
    }
    return []int(*h)
}
```

**Сложность:** O(n log k) — лучше сортировки O(n log n) при k << n.

### Merge K отсортированных списков

Min-heap из K элементов (по одному из каждого списка). Извлекай min, добавляй следующий из того же списка.

**Сложность:** O(N log K), где N — общее количество элементов.

### Медиана потока данных

Два heap'а: max-heap для левой половины, min-heap для правой.

```
Max-heap (левая): [1, 2, 3]  ← max = 3
Min-heap (правая): [4, 5, 6] ← min = 4
Медиана = (3 + 4) / 2 = 3.5
```

Балансируй размеры: разница ≤ 1.

## Heap Sort

```go
// 1. Построить max-heap из массива: O(n)
// 2. Извлекать max и ставить в конец: O(n log n)
// Итого: O(n log n), in-place, не стабильная
```

Практически уступает quicksort из-за плохой cache locality, но гарантирует O(n log n) в worst case.
