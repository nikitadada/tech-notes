# Sliding Window

Окно фиксированного или переменного размера двигается по массиву/строке. Эффективно обрабатывает подмассивы/подстроки за O(n).

## Фиксированное окно

Размер окна задан. При сдвиге: добавляем новый элемент, убираем старый.

**Пример: максимальная средняя подмассива длины k**

```go
func findMaxAverage(nums []int, k int) float64 {
    // Начальная сумма окна
    windowSum := 0
    for i := 0; i < k; i++ {
        windowSum += nums[i]
    }
    maxSum := windowSum

    // Сдвигаем окно
    for i := k; i < len(nums); i++ {
        windowSum += nums[i] - nums[i-k]  // добавить новый, убрать старый
        maxSum = max(maxSum, windowSum)
    }
    return float64(maxSum) / float64(k)
}
```

## Переменное окно

Размер окна меняется динамически. Правая граница расширяет, левая сужает.

### Шаблон

```go
func slidingWindow(s string) int {
    left := 0
    best := 0
    state := ... // hash map, counter, etc.

    for right := 0; right < len(s); right++ {
        // 1. Расширяем окно: добавляем s[right] в state

        // 2. Сужаем окно пока условие нарушено
        for /* условие нарушено */ {
            // убираем s[left] из state
            left++
        }

        // 3. Обновляем результат
        best = max(best, right-left+1)
    }
    return best
}
```

### Пример: Longest Substring Without Repeating Characters

```go
func lengthOfLongestSubstring(s string) int {
    seen := make(map[byte]int) // символ → последний индекс
    left := 0
    best := 0

    for right := 0; right < len(s); right++ {
        if idx, ok := seen[s[right]]; ok && idx >= left {
            left = idx + 1
        }
        seen[s[right]] = right
        best = max(best, right-left+1)
    }
    return best
}
```

### Пример: Minimum Size Subarray Sum

Минимальная длина подмассива с суммой ≥ target.

```go
func minSubArrayLen(target int, nums []int) int {
    left := 0
    sum := 0
    best := len(nums) + 1

    for right := 0; right < len(nums); right++ {
        sum += nums[right]

        for sum >= target {
            best = min(best, right-left+1)
            sum -= nums[left]
            left++
        }
    }

    if best == len(nums)+1 {
        return 0
    }
    return best
}
```

### Пример: Minimum Window Substring

Найти минимальную подстроку s, содержащую все символы t.

```go
func minWindow(s, t string) string {
    need := make(map[byte]int)
    for i := range t {
        need[t[i]]++
    }

    window := make(map[byte]int)
    have, total := 0, len(need)
    left := 0
    bestLen := len(s) + 1
    bestStart := 0

    for right := 0; right < len(s); right++ {
        ch := s[right]
        window[ch]++
        if window[ch] == need[ch] {
            have++
        }

        for have == total {
            if right-left+1 < bestLen {
                bestLen = right - left + 1
                bestStart = left
            }
            lch := s[left]
            window[lch]--
            if window[lch] < need[lch] {
                have--
            }
            left++
        }
    }

    if bestLen == len(s)+1 {
        return ""
    }
    return s[bestStart : bestStart+bestLen]
}
```

## Когда применять

- "Найти подмассив/подстроку с условием" → sliding window
- Фиксированная длина → фиксированное окно
- Минимальная/максимальная длина с условием → переменное окно
- Ключевые слова: contiguous, subarray, substring
