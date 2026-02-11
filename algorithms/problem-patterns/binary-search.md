# Binary Search

Поиск в отсортированном пространстве за O(log n). Каждый шаг делит пространство пополам.

## Базовый шаблон

```go
func binarySearch(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2  // избегаем overflow
        switch {
        case nums[mid] == target:
            return mid
        case nums[mid] < target:
            lo = mid + 1
        default:
            hi = mid - 1
        }
    }
    return -1 // не найден
}
```

## Поиск границ

### Левая граница (первый >= target)

```go
func lowerBound(nums []int, target int) int {
    lo, hi := 0, len(nums)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if nums[mid] < target {
            lo = mid + 1
        } else {
            hi = mid
        }
    }
    return lo
}
```

### Правая граница (последний <= target)

```go
func upperBound(nums []int, target int) int {
    lo, hi := 0, len(nums)
    for lo < hi {
        mid := lo + (hi-lo)/2
        if nums[mid] <= target {
            lo = mid + 1
        } else {
            hi = mid
        }
    }
    return lo - 1
}
```

### Go стандартная библиотека

```go
import "sort"

// Первый индекс где nums[i] >= target
idx := sort.SearchInts(nums, target)

// Универсальный: первый индекс где f(i) == true
idx := sort.Search(len(nums), func(i int) bool {
    return nums[i] >= target
})
```

## Binary Search на ответ

Когда ответ монотонен: "можем ли мы достичь X?" → binary search по X.

### Пример: Koko Eating Bananas

Минимальная скорость k, чтобы съесть все бананы за h часов.

```go
func minEatingSpeed(piles []int, h int) int {
    lo, hi := 1, maxElement(piles)

    for lo < hi {
        mid := lo + (hi-lo)/2
        if canFinish(piles, mid, h) {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}

func canFinish(piles []int, speed, hours int) bool {
    total := 0
    for _, p := range piles {
        total += (p + speed - 1) / speed // ceil division
    }
    return total <= hours
}
```

### Пример: Split Array Largest Sum

Разделить массив на m частей, минимизируя максимальную сумму части.

```go
func splitArray(nums []int, m int) int {
    lo, hi := maxElement(nums), sum(nums)

    for lo < hi {
        mid := lo + (hi-lo)/2
        if canSplit(nums, m, mid) {
            hi = mid
        } else {
            lo = mid + 1
        }
    }
    return lo
}

func canSplit(nums []int, m, maxSum int) bool {
    parts, currSum := 1, 0
    for _, n := range nums {
        if currSum+n > maxSum {
            parts++
            currSum = n
        } else {
            currSum += n
        }
    }
    return parts <= m
}
```

## В ротированном массиве

```go
func searchRotated(nums []int, target int) int {
    lo, hi := 0, len(nums)-1
    for lo <= hi {
        mid := lo + (hi-lo)/2
        if nums[mid] == target {
            return mid
        }
        if nums[lo] <= nums[mid] { // левая часть отсортирована
            if nums[lo] <= target && target < nums[mid] {
                hi = mid - 1
            } else {
                lo = mid + 1
            }
        } else { // правая часть отсортирована
            if nums[mid] < target && target <= nums[hi] {
                lo = mid + 1
            } else {
                hi = mid - 1
            }
        }
    }
    return -1
}
```

## Когда применять

- Отсортированный массив → классический binary search
- "Найти минимальное/максимальное значение, удовлетворяющее условию" → binary search на ответ
- Монотонная функция f(x): false...false, true...true → поиск первого true
- Ключевые слова: minimum, maximum, sorted, capacity, speed
