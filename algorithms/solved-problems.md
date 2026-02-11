# Разобранные задачи

## Two Sum

**Задача:** Найти два числа в массиве, дающих в сумме target. Вернуть их индексы.

**Подход:** Hash map — за один проход запоминаем complement.

```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int) // value → index
    for i, n := range nums {
        if j, ok := seen[target-n]; ok {
            return []int{j, i}
        }
        seen[n] = i
    }
    return nil
}
```

**Сложность:** Time O(n), Space O(n)

---

## Valid Parentheses

**Задача:** Проверить, что строка из скобок `()[]{}` корректно вложена.

**Подход:** Стек — открывающие кладём, закрывающие сверяем с вершиной.

```go
func isValid(s string) bool {
    stack := []rune{}
    pairs := map[rune]rune{')': '(', ']': '[', '}': '{'}

    for _, ch := range s {
        if match, isClosing := pairs[ch]; isClosing {
            if len(stack) == 0 || stack[len(stack)-1] != match {
                return false
            }
            stack = stack[:len(stack)-1]
        } else {
            stack = append(stack, ch)
        }
    }
    return len(stack) == 0
}
```

**Сложность:** Time O(n), Space O(n)

---

## Merge Intervals

**Задача:** Объединить пересекающиеся интервалы.

**Подход:** Сортировка по началу + merge.

```go
func merge(intervals [][]int) [][]int {
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][0] < intervals[j][0]
    })

    result := [][]int{intervals[0]}
    for _, curr := range intervals[1:] {
        last := result[len(result)-1]
        if curr[0] <= last[1] {
            last[1] = max(last[1], curr[1])
        } else {
            result = append(result, curr)
        }
    }
    return result
}
```

**Сложность:** Time O(n log n), Space O(n)

---

## Binary Search (шаблон)

```go
func binarySearch(lo, hi int, predicate func(mid int) bool) int {
    for lo < hi {
        mid := lo + (hi-lo)/2
        if predicate(mid) {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}
```

Универсальный шаблон: ищет первый элемент, для которого predicate == true.

---

## LRU Cache

**Задача:** Реализовать кэш с вытеснением наименее используемых элементов.

**Подход:** Hash map + doubly linked list.

```go
type LRUCache struct {
    capacity int
    cache    map[int]*list.Element
    order    *list.List
}

type entry struct {
    key, value int
}

func Constructor(capacity int) LRUCache {
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*list.Element),
        order:    list.New(),
    }
}

func (c *LRUCache) Get(key int) int {
    if elem, ok := c.cache[key]; ok {
        c.order.MoveToFront(elem)
        return elem.Value.(*entry).value
    }
    return -1
}

func (c *LRUCache) Put(key, value int) {
    if elem, ok := c.cache[key]; ok {
        elem.Value.(*entry).value = value
        c.order.MoveToFront(elem)
        return
    }
    if c.order.Len() >= c.capacity {
        oldest := c.order.Back()
        c.order.Remove(oldest)
        delete(c.cache, oldest.Value.(*entry).key)
    }
    elem := c.order.PushFront(&entry{key, value})
    c.cache[key] = elem
}
```

**Сложность:** Get O(1), Put O(1), Space O(capacity)

---

## Шаблон для добавления новых задач

```
## Название задачи

**Задача:** ...
**Подход:** ...
**Код:** ...
**Сложность:** Time O(?), Space O(?)
**Заметки:** ...
```
