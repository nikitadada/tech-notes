# Backtracking

Построение решения пошагово с откатом (backtrack) при обнаружении тупика. По сути — DFS по дереву решений с pruning.

## Шаблон

```go
func backtrack(state State, choices []Choice, result *[]Solution) {
    if isComplete(state) {
        result = append(result, copy(state))
        return
    }

    for _, choice := range choices {
        if !isValid(state, choice) {
            continue // pruning
        }

        // 1. Сделать выбор
        apply(state, choice)

        // 2. Рекурсия
        backtrack(state, nextChoices, result)

        // 3. Отменить выбор (backtrack)
        undo(state, choice)
    }
}
```

## Классические задачи

### Subsets (все подмножества)

```go
func subsets(nums []int) [][]int {
    var result [][]int
    var current []int

    var bt func(start int)
    bt = func(start int) {
        // Каждое промежуточное состояние — валидное подмножество
        cp := make([]int, len(current))
        copy(cp, current)
        result = append(result, cp)

        for i := start; i < len(nums); i++ {
            current = append(current, nums[i])
            bt(i + 1)
            current = current[:len(current)-1]
        }
    }

    bt(0)
    return result
}
```

### Permutations (все перестановки)

```go
func permute(nums []int) [][]int {
    var result [][]int
    used := make([]bool, len(nums))
    var current []int

    var bt func()
    bt = func() {
        if len(current) == len(nums) {
            cp := make([]int, len(current))
            copy(cp, current)
            result = append(result, cp)
            return
        }

        for i := 0; i < len(nums); i++ {
            if used[i] {
                continue
            }
            used[i] = true
            current = append(current, nums[i])
            bt()
            current = current[:len(current)-1]
            used[i] = false
        }
    }

    bt()
    return result
}
```

### Combination Sum

Найти все комбинации чисел, дающих в сумме target (элемент можно использовать повторно).

```go
func combinationSum(candidates []int, target int) [][]int {
    var result [][]int
    var current []int

    var bt func(start, remaining int)
    bt = func(start, remaining int) {
        if remaining == 0 {
            cp := make([]int, len(current))
            copy(cp, current)
            result = append(result, cp)
            return
        }

        for i := start; i < len(candidates); i++ {
            if candidates[i] > remaining {
                continue // pruning
            }
            current = append(current, candidates[i])
            bt(i, remaining-candidates[i]) // i, не i+1 — можно повторять
            current = current[:len(current)-1]
        }
    }

    sort.Ints(candidates)
    bt(0, target)
    return result
}
```

### N-Queens

Расставить N ферзей на N×N доске без взаимных атак.

```go
func solveNQueens(n int) [][]string {
    var result [][]string
    board := make([]int, n) // board[row] = col

    cols := make([]bool, n)
    diag1 := make([]bool, 2*n) // row - col + n
    diag2 := make([]bool, 2*n) // row + col

    var bt func(row int)
    bt = func(row int) {
        if row == n {
            result = append(result, formatBoard(board, n))
            return
        }

        for col := 0; col < n; col++ {
            if cols[col] || diag1[row-col+n] || diag2[row+col] {
                continue
            }
            board[row] = col
            cols[col] = true
            diag1[row-col+n] = true
            diag2[row+col] = true

            bt(row + 1)

            cols[col] = false
            diag1[row-col+n] = false
            diag2[row+col] = false
        }
    }

    bt(0)
    return result
}
```

### Word Search

Найти слово в матрице символов (ходы в 4 направлениях, каждая клетка один раз).

```go
func exist(board [][]byte, word string) bool {
    rows, cols := len(board), len(board[0])

    var bt func(r, c, idx int) bool
    bt = func(r, c, idx int) bool {
        if idx == len(word) {
            return true
        }
        if r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] != word[idx] {
            return false
        }

        ch := board[r][c]
        board[r][c] = '#' // mark visited

        found := bt(r+1, c, idx+1) || bt(r-1, c, idx+1) ||
                 bt(r, c+1, idx+1) || bt(r, c-1, idx+1)

        board[r][c] = ch // restore
        return found
    }

    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if bt(r, c, 0) {
                return true
            }
        }
    }
    return false
}
```

## Pruning (отсечение)

Ключ к эффективному backtracking. Отсекай заведомо тупиковые ветви как можно раньше:
- Сумма превышает target → не продолжай
- Конфликт на доске → не ставь фигуру
- Оставшихся элементов не хватит → stop

## Когда применять

- "Найди все..." → backtracking
- "Сколько вариантов..." → backtracking (или DP если нужно только количество)
- Перестановки, комбинации, подмножества
- Constraint satisfaction (N-Queens, Sudoku)
