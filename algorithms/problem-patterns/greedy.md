# Greedy

На каждом шаге выбираем локально оптимальное решение, надеясь получить глобально оптимальное.

## Когда работает

- Задача имеет **greedy choice property** — локально оптимальный выбор ведёт к глобальному оптимуму
- Задача имеет **оптимальную подструктуру** — оптимальное решение содержит оптимальные решения подзадач

## Отличие от DP

| Критерий | Greedy | DP |
|----------|--------|-----|
| Выбор | Локально оптимальный | Рассматривает все варианты |
| Возврат | Нет (необратимый выбор) | Да (через подзадачи) |
| Сложность | Обычно O(n log n) | Обычно O(n²) или O(n * W) |
| Гарантия оптимальности | Только для определённых задач | Всегда (при правильной формулировке) |

## Классические задачи

### Activity Selection (Interval Scheduling)

Максимальное количество непересекающихся интервалов.

**Стратегия:** сортируй по концу, бери первый доступный.

```go
func maxActivities(intervals [][]int) int {
    sort.Slice(intervals, func(i, j int) bool {
        return intervals[i][1] < intervals[j][1]
    })

    count := 1
    end := intervals[0][1]

    for i := 1; i < len(intervals); i++ {
        if intervals[i][0] >= end {
            count++
            end = intervals[i][1]
        }
    }
    return count
}
```

### Jump Game

Можно ли добраться до конца массива. `nums[i]` — максимальный прыжок из позиции i.

```go
func canJump(nums []int) bool {
    maxReach := 0
    for i := 0; i < len(nums); i++ {
        if i > maxReach {
            return false
        }
        maxReach = max(maxReach, i+nums[i])
    }
    return true
}
```

### Jump Game II (минимум прыжков)

```go
func jump(nums []int) int {
    jumps := 0
    currEnd := 0
    farthest := 0

    for i := 0; i < len(nums)-1; i++ {
        farthest = max(farthest, i+nums[i])
        if i == currEnd {
            jumps++
            currEnd = farthest
        }
    }
    return jumps
}
```

### Gas Station

Круговой маршрут с заправками. Найти стартовую позицию.

```go
func canCompleteCircuit(gas, cost []int) int {
    totalSurplus := 0
    currentSurplus := 0
    start := 0

    for i := 0; i < len(gas); i++ {
        totalSurplus += gas[i] - cost[i]
        currentSurplus += gas[i] - cost[i]
        if currentSurplus < 0 {
            start = i + 1
            currentSurplus = 0
        }
    }

    if totalSurplus < 0 {
        return -1
    }
    return start
}
```

### Task Scheduler

Минимальное время выполнения задач с cooldown между одинаковыми.

```go
func leastInterval(tasks []byte, n int) int {
    freq := make([]int, 26)
    for _, t := range tasks {
        freq[t-'A']++
    }
    sort.Ints(freq)

    maxFreq := freq[25]
    // Количество задач с максимальной частотой
    maxCount := 0
    for _, f := range freq {
        if f == maxFreq {
            maxCount++
        }
    }

    // (maxFreq-1) полных блоков размера (n+1) + хвост
    result := (maxFreq-1)*(n+1) + maxCount
    return max(result, len(tasks))
}
```

## Стратегии greedy

| Стратегия | Задачи |
|-----------|--------|
| Сортируй по концу | Interval scheduling, meeting rooms |
| Бери наибольший/наименьший | Fractional knapsack, assign cookies |
| Отслеживай максимум/минимум | Jump game, gas station |
| Считай баланс | Valid parentheses (greedy), task scheduler |

## Как доказать корректность

1. **Exchange argument:** покажи, что замена greedy выбора на другой не улучшает результат
2. **Staying ahead:** покажи, что greedy решение не хуже оптимального на каждом шаге
