# Linked Lists

Элементы хранятся в отдельных узлах, связанных указателями. Нет непрерывного блока памяти.

## Виды

| Вид | Описание |
|-----|----------|
| Singly linked | Каждый узел → следующий |
| Doubly linked | Каждый узел ↔ предыдущий и следующий |
| Circular | Последний узел → первый |

## Сложность

| Операция | Singly | Doubly |
|----------|--------|--------|
| Доступ по индексу | O(n) | O(n) |
| Вставка в начало | O(1) | O(1) |
| Вставка в конец | O(n) / O(1)* | O(1) |
| Удаление по указателю | O(n) | O(1) |
| Поиск | O(n) | O(n) |

*O(1) если хранить указатель на хвост

## Реализация (singly linked list)

```go
type Node struct {
    Val  int
    Next *Node
}

type LinkedList struct {
    Head *Node
    Size int
}

func (l *LinkedList) PushFront(val int) {
    l.Head = &Node{Val: val, Next: l.Head}
    l.Size++
}

func (l *LinkedList) PopFront() (int, bool) {
    if l.Head == nil {
        return 0, false
    }
    val := l.Head.Val
    l.Head = l.Head.Next
    l.Size--
    return val, true
}
```

## Типичные приёмы

### Dummy node (sentinel)

Упрощает edge cases (пустой список, удаление head).

```go
func removeElements(head *Node, val int) *Node {
    dummy := &Node{Next: head}
    curr := dummy
    for curr.Next != nil {
        if curr.Next.Val == val {
            curr.Next = curr.Next.Next
        } else {
            curr = curr.Next
        }
    }
    return dummy.Next
}
```

### Быстрый и медленный указатель

Нахождение середины списка, обнаружение цикла.

```go
// Середина списка
func middleNode(head *Node) *Node {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    return slow
}

// Обнаружение цикла (Floyd's algorithm)
func hasCycle(head *Node) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast {
            return true
        }
    }
    return false
}
```

### Разворот списка

```go
func reverseList(head *Node) *Node {
    var prev *Node
    curr := head
    for curr != nil {
        next := curr.Next
        curr.Next = prev
        prev = curr
        curr = next
    }
    return prev
}
```

### Слияние двух отсортированных списков

```go
func mergeTwoLists(l1, l2 *Node) *Node {
    dummy := &Node{}
    curr := dummy
    for l1 != nil && l2 != nil {
        if l1.Val <= l2.Val {
            curr.Next = l1
            l1 = l1.Next
        } else {
            curr.Next = l2
            l2 = l2.Next
        }
        curr = curr.Next
    }
    if l1 != nil {
        curr.Next = l1
    } else {
        curr.Next = l2
    }
    return dummy.Next
}
```

## Когда использовать

- Частые вставки/удаления в начало — linked list лучше array
- LRU Cache — doubly linked list + hash map
- Реализация стека/очереди
- На практике в Go чаще используются slices из-за cache locality
