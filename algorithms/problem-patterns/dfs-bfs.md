# DFS & BFS Patterns

## Когда DFS, когда BFS

| Критерий | DFS | BFS |
|----------|-----|-----|
| Кратчайший путь (невзвешенный) | Нет | Да |
| Обход всех путей | Да | Нет (дорого) |
| Компоненты связности | Да | Да |
| Топологическая сортировка | Да | Да (Kahn's) |
| Уровни/расстояния | Нет | Да |
| Память | O(h) — глубина | O(w) — ширина уровня |

## DFS на деревьях

### Шаблон рекурсивного DFS

```go
func dfs(root *TreeNode) int {
    if root == nil {
        return baseCase
    }
    left := dfs(root.Left)
    right := dfs(root.Right)
    return combine(left, right, root.Val)
}
```

### Максимальная глубина дерева

```go
func maxDepth(root *TreeNode) int {
    if root == nil {
        return 0
    }
    return 1 + max(maxDepth(root.Left), maxDepth(root.Right))
}
```

### Path Sum — существует ли путь с данной суммой

```go
func hasPathSum(root *TreeNode, target int) bool {
    if root == nil {
        return false
    }
    if root.Left == nil && root.Right == nil {
        return root.Val == target
    }
    remaining := target - root.Val
    return hasPathSum(root.Left, remaining) || hasPathSum(root.Right, remaining)
}
```

## DFS на графах / матрицах

### Подсчёт островов (Number of Islands)

```go
func numIslands(grid [][]byte) int {
    rows, cols := len(grid), len(grid[0])
    count := 0

    var dfs func(r, c int)
    dfs = func(r, c int) {
        if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] != '1' {
            return
        }
        grid[r][c] = '0' // mark visited
        dfs(r+1, c)
        dfs(r-1, c)
        dfs(r, c+1)
        dfs(r, c-1)
    }

    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if grid[r][c] == '1' {
                count++
                dfs(r, c)
            }
        }
    }
    return count
}
```

## BFS — кратчайший путь

### Шаблон BFS

```go
func bfs(start Node) int {
    queue := []Node{start}
    visited := map[Node]bool{start: true}
    level := 0

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            node := queue[0]
            queue = queue[1:]

            if isTarget(node) {
                return level
            }

            for _, next := range neighbors(node) {
                if !visited[next] {
                    visited[next] = true
                    queue = append(queue, next)
                }
            }
        }
        level++
    }
    return -1 // не найден
}
```

### Кратчайший путь в лабиринте

```go
type Point struct{ r, c int }

var dirs = []Point{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}

func shortestPath(grid [][]int, start, end Point) int {
    rows, cols := len(grid), len(grid[0])
    visited := make([][]bool, rows)
    for i := range visited {
        visited[i] = make([]bool, cols)
    }

    queue := []Point{start}
    visited[start.r][start.c] = true
    steps := 0

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            p := queue[0]
            queue = queue[1:]

            if p == end {
                return steps
            }

            for _, d := range dirs {
                nr, nc := p.r+d.r, p.c+d.c
                if nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
                    !visited[nr][nc] && grid[nr][nc] == 0 {
                    visited[nr][nc] = true
                    queue = append(queue, Point{nr, nc})
                }
            }
        }
        steps++
    }
    return -1
}
```

## Multi-source BFS

Стартуем BFS одновременно из нескольких источников. Полезно для задач "расстояние до ближайшего X".

```go
// Все '0' в очередь → BFS → расстояние до ближайшего '0' для каждой клетки
func updateMatrix(mat [][]int) [][]int {
    rows, cols := len(mat), len(mat[0])
    dist := make([][]int, rows)
    queue := []Point{}

    for r := 0; r < rows; r++ {
        dist[r] = make([]int, cols)
        for c := 0; c < cols; c++ {
            if mat[r][c] == 0 {
                queue = append(queue, Point{r, c})
            } else {
                dist[r][c] = math.MaxInt32
            }
        }
    }

    for len(queue) > 0 {
        p := queue[0]
        queue = queue[1:]
        for _, d := range dirs {
            nr, nc := p.r+d.r, p.c+d.c
            if nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
                dist[nr][nc] > dist[p.r][p.c]+1 {
                dist[nr][nc] = dist[p.r][p.c] + 1
                queue = append(queue, Point{nr, nc})
            }
        }
    }
    return dist
}
```

## Подсказки

- Кратчайший путь → BFS
- "Все пути" / "существует ли путь" → DFS
- Матрица как граф → каждая клетка — вершина, соседи — 4 направления
- Не забывай `visited` — иначе бесконечный цикл
