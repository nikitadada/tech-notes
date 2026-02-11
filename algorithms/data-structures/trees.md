# Trees

Иерархическая структура данных: узлы с дочерними элементами. Корень → ... → листья.

## Binary Tree

Каждый узел имеет максимум двух детей.

```go
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}
```

## Обходы

```
        4
       / \
      2   6
     / \ / \
    1  3 5  7
```

| Обход | Порядок | Результат | Применение |
|-------|---------|-----------|------------|
| Inorder (LNR) | Лево → Узел → Право | 1,2,3,4,5,6,7 | Отсортированный вывод BST |
| Preorder (NLR) | Узел → Лево → Право | 4,2,1,3,6,5,7 | Сериализация дерева |
| Postorder (LRN) | Лево → Право → Узел | 1,3,2,5,7,6,4 | Удаление дерева, вычисление выражений |
| Level order (BFS) | Уровень за уровнем | 4,2,6,1,3,5,7 | Поиск ближайшего узла |

```go
// Inorder (рекурсивный)
func inorder(root *TreeNode, result *[]int) {
    if root == nil {
        return
    }
    inorder(root.Left, result)
    *result = append(*result, root.Val)
    inorder(root.Right, result)
}

// Level order (BFS)
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return nil
    }
    var result [][]int
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        size := len(queue)
        level := make([]int, 0, size)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Val)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}
```

## BST (Binary Search Tree)

Для каждого узла: все значения слева < узел < все значения справа.

| Операция | Средняя | Worst case (несбалансированное) |
|----------|---------|-------------------------------|
| Поиск | O(log n) | O(n) |
| Вставка | O(log n) | O(n) |
| Удаление | O(log n) | O(n) |

```go
func search(root *TreeNode, target int) *TreeNode {
    if root == nil || root.Val == target {
        return root
    }
    if target < root.Val {
        return search(root.Left, target)
    }
    return search(root.Right, target)
}

func insert(root *TreeNode, val int) *TreeNode {
    if root == nil {
        return &TreeNode{Val: val}
    }
    if val < root.Val {
        root.Left = insert(root.Left, val)
    } else {
        root.Right = insert(root.Right, val)
    }
    return root
}
```

## Самобалансирующиеся деревья

Гарантируют O(log n) для всех операций.

| Дерево | Описание |
|--------|----------|
| AVL | Разница высот поддеревьев ≤ 1. Строгий баланс, быстрый поиск |
| Red-Black | Менее строгий баланс, быстрая вставка/удаление. Используется в `std::map` |
| B-Tree | Многоветвистое. Используется в БД и файловых системах |

## Trie (префиксное дерево)

Для хранения строк с общими префиксами. Поиск за O(m), где m — длина строки.

```go
type TrieNode struct {
    Children map[rune]*TrieNode
    IsEnd    bool
}

type Trie struct {
    Root *TrieNode
}

func NewTrie() *Trie {
    return &Trie{Root: &TrieNode{Children: make(map[rune]*TrieNode)}}
}

func (t *Trie) Insert(word string) {
    node := t.Root
    for _, ch := range word {
        if _, ok := node.Children[ch]; !ok {
            node.Children[ch] = &TrieNode{Children: make(map[rune]*TrieNode)}
        }
        node = node.Children[ch]
    }
    node.IsEnd = true
}

func (t *Trie) Search(word string) bool {
    node := t.Root
    for _, ch := range word {
        if _, ok := node.Children[ch]; !ok {
            return false
        }
        node = node.Children[ch]
    }
    return node.IsEnd
}
```

**Применение:** автодополнение, spell-check, IP routing, словари.

## Типичные задачи

- Максимальная глубина: рекурсивно `max(depth(left), depth(right)) + 1`
- Проверка сбалансированности: глубины поддеревьев отличаются ≤ 1
- LCA (Lowest Common Ancestor): для BST — первый узел между p и q
- Сериализация/десериализация: preorder + null markers
