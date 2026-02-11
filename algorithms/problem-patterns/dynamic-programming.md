# Dynamic Programming (DP)

Разбиение задачи на перекрывающиеся подзадачи. Запоминание результатов подзадач для избежания повторных вычислений.

## Признаки DP-задачи

1. **Оптимальная подструктура** — оптимальное решение содержит оптимальные решения подзадач
2. **Перекрывающиеся подзадачи** — одни и те же подзадачи решаются многократно
3. Ключевые слова: "минимальное количество", "сколько способов", "максимальная прибыль", "можно ли"

## Подходы

### Top-Down (мемоизация)

Рекурсия + кэш результатов.

```go
func fib(n int, memo map[int]int) int {
    if n <= 1 {
        return n
    }
    if v, ok := memo[n]; ok {
        return v
    }
    memo[n] = fib(n-1, memo) + fib(n-2, memo)
    return memo[n]
}
```

### Bottom-Up (табуляция)

Итеративно заполняем таблицу от простых подзадач к сложным.

```go
func fib(n int) int {
    if n <= 1 {
        return n
    }
    dp := make([]int, n+1)
    dp[0], dp[1] = 0, 1
    for i := 2; i <= n; i++ {
        dp[i] = dp[i-1] + dp[i-2]
    }
    return dp[n]
}

// Оптимизация памяти: O(1) вместо O(n)
func fib(n int) int {
    if n <= 1 {
        return n
    }
    prev, curr := 0, 1
    for i := 2; i <= n; i++ {
        prev, curr = curr, prev+curr
    }
    return curr
}
```

## Классические задачи

### Climbing Stairs

Сколько способов подняться на n ступенек (1 или 2 за раз).

```go
// dp[i] = dp[i-1] + dp[i-2]  (Fibonacci)
func climbStairs(n int) int {
    if n <= 2 {
        return n
    }
    prev, curr := 1, 2
    for i := 3; i <= n; i++ {
        prev, curr = curr, prev+curr
    }
    return curr
}
```

### Coin Change

Минимальное количество монет для суммы amount.

```go
// dp[i] = min(dp[i - coin] + 1) для каждой монеты
func coinChange(coins []int, amount int) int {
    dp := make([]int, amount+1)
    for i := range dp {
        dp[i] = amount + 1 // "бесконечность"
    }
    dp[0] = 0

    for i := 1; i <= amount; i++ {
        for _, coin := range coins {
            if coin <= i && dp[i-coin]+1 < dp[i] {
                dp[i] = dp[i-coin] + 1
            }
        }
    }

    if dp[amount] > amount {
        return -1
    }
    return dp[amount]
}
```

### Longest Common Subsequence (LCS)

```go
// dp[i][j] = LCS длины text1[:i] и text2[:j]
func longestCommonSubsequence(text1, text2 string) int {
    m, n := len(text1), len(text2)
    dp := make([][]int, m+1)
    for i := range dp {
        dp[i] = make([]int, n+1)
    }

    for i := 1; i <= m; i++ {
        for j := 1; j <= n; j++ {
            if text1[i-1] == text2[j-1] {
                dp[i][j] = dp[i-1][j-1] + 1
            } else {
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
            }
        }
    }
    return dp[m][n]
}
```

### 0/1 Knapsack

```go
// dp[i][w] = макс. ценность из первых i предметов при вместимости w
func knapsack(weights, values []int, capacity int) int {
    n := len(weights)
    dp := make([][]int, n+1)
    for i := range dp {
        dp[i] = make([]int, capacity+1)
    }

    for i := 1; i <= n; i++ {
        for w := 0; w <= capacity; w++ {
            dp[i][w] = dp[i-1][w] // не берём
            if weights[i-1] <= w {
                dp[i][w] = max(dp[i][w], dp[i-1][w-weights[i-1]]+values[i-1])
            }
        }
    }
    return dp[n][capacity]
}
```

## Фреймворк решения

1. **Определи состояние** — что описывает подзадачу? (индекс, оставшаяся сумма, etc.)
2. **Рекуррентное соотношение** — как связаны подзадачи?
3. **Базовый случай** — тривиальные подзадачи
4. **Порядок вычисления** — от простого к сложному
5. **Оптимизация памяти** — если dp[i] зависит только от dp[i-1]

## Категории DP-задач

| Категория | Примеры |
|-----------|---------|
| 1D linear | Climbing stairs, House robber, Coin change |
| 2D grid | Unique paths, Minimum path sum |
| String | LCS, Edit distance, Palindrome |
| Knapsack | 0/1 knapsack, Subset sum, Partition equal |
| Interval | Matrix chain, Burst balloons |
| Tree DP | House robber III, Diameter of tree |
