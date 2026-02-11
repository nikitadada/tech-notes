# Graphs

Набор вершин (vertices) и рёбер (edges) между ними.

## Представления

### Adjacency List

```go
// map[вершина][]соседи
graph := map[int][]int{
    0: {1, 2},
    1: {0, 3},
    2: {0, 3},
    3: {1, 2},
}
```

- Память: O(V + E)
- Проверка ребра: O(degree)
- Хорошо для разреженных графов

### Adjacency Matrix

```go
// matrix[i][j] = 1 если есть ребро
matrix := [][]int{
    {0, 1, 1, 0},
    {1, 0, 0, 1},
    {1, 0, 0, 1},
    {0, 1, 1, 0},
}
```

- Память: O(V²)
- Проверка ребра: O(1)
- Хорошо для плотных графов

## DFS (Depth-First Search)

Идёт вглубь, использует стек (или рекурсию).

```go
func dfs(graph map[int][]int, start int) []int {
    visited := make(map[int]bool)
    var result []int

    var visit func(node int)
    visit = func(node int) {
        if visited[node] {
            return
        }
        visited[node] = true
        result = append(result, node)
        for _, neighbor := range graph[node] {
            visit(neighbor)
        }
    }

    visit(start)
    return result
}
```

**Применение:** поиск пути, обнаружение цикла, топологическая сортировка, компоненты связности.

## BFS (Breadth-First Search)

Идёт вширь, использует очередь. Находит кратчайший путь в невзвешенном графе.

```go
func bfs(graph map[int][]int, start int) []int {
    visited := map[int]bool{start: true}
    queue := []int{start}
    var result []int

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        result = append(result, node)

        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }
    return result
}
```

**Применение:** кратчайший путь (невзвешенный), уровни в графе, поиск ближайшего.

## Топологическая сортировка

Линейный порядок вершин DAG (directed acyclic graph), где для каждого ребра u→v вершина u идёт раньше v.

```go
// Kahn's algorithm (BFS-based)
func topologicalSort(numNodes int, edges [][]int) []int {
    inDegree := make([]int, numNodes)
    adj := make([][]int, numNodes)

    for _, e := range edges {
        adj[e[0]] = append(adj[e[0]], e[1])
        inDegree[e[1]]++
    }

    queue := []int{}
    for i, d := range inDegree {
        if d == 0 {
            queue = append(queue, i)
        }
    }

    var order []int
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        order = append(order, node)
        for _, next := range adj[node] {
            inDegree[next]--
            if inDegree[next] == 0 {
                queue = append(queue, next)
            }
        }
    }

    if len(order) != numNodes {
        return nil // есть цикл
    }
    return order
}
```

**Применение:** порядок сборки (build systems), расписание задач, разрешение зависимостей.

## Dijkstra (кратчайший путь во взвешенном графе)

```go
func dijkstra(graph map[int][]Edge, start int) map[int]int {
    dist := map[int]int{start: 0}
    // min-heap: (distance, node)
    pq := &PriorityQueue{}
    heap.Push(pq, Item{dist: 0, node: start})

    for pq.Len() > 0 {
        curr := heap.Pop(pq).(Item)
        if d, ok := dist[curr.node]; ok && curr.dist > d {
            continue // уже нашли лучший путь
        }
        for _, edge := range graph[curr.node] {
            newDist := curr.dist + edge.Weight
            if d, ok := dist[edge.To]; !ok || newDist < d {
                dist[edge.To] = newDist
                heap.Push(pq, Item{dist: newDist, node: edge.To})
            }
        }
    }
    return dist
}
```

**Сложность:** O((V + E) log V) с binary heap.
**Ограничение:** не работает с отрицательными весами (используй Bellman-Ford).

## Обнаружение цикла

### В ненаправленном графе

DFS: если сосед уже visited и не parent → цикл.

### В направленном графе

DFS с тремя состояниями: white (не посещён), gray (в обработке), black (завершён). Ребро в gray вершину → цикл.

```go
func hasCycle(graph map[int][]int) bool {
    color := make(map[int]int) // 0=white, 1=gray, 2=black

    var dfs func(node int) bool
    dfs = func(node int) bool {
        color[node] = 1
        for _, next := range graph[node] {
            if color[next] == 1 {
                return true // back edge → цикл
            }
            if color[next] == 0 && dfs(next) {
                return true
            }
        }
        color[node] = 2
        return false
    }

    for node := range graph {
        if color[node] == 0 && dfs(node) {
            return true
        }
    }
    return false
}
```
