# Arrays & Strings

## Массив

Непрерывный блок памяти с O(1) доступом по индексу.

| Операция | Сложность |
|----------|-----------|
| Доступ по индексу | O(1) |
| Поиск | O(n) |
| Вставка в конец | O(1) amortized |
| Вставка в середину | O(n) |
| Удаление из середины | O(n) |

### Slice в Go

```go
// Создание
s := make([]int, 0, 10)  // len=0, cap=10
s = append(s, 1, 2, 3)

// Внутреннее устройство
// slice header: { ptr *array, len int, cap int }
// При append и cap исчерпан → аллокация нового массива (x2 до 256, потом ~1.25x)
```

**Подводные камни:**

```go
// Slice ссылается на underlying array
a := []int{1, 2, 3, 4, 5}
b := a[1:3]  // [2, 3] — разделяет память с a
b[0] = 99    // a = [1, 99, 3, 4, 5]

// Безопасное копирование
b := make([]int, len(a))
copy(b, a)
```

## Строки в Go

Строка — immutable byte slice. Итерация по `range` даёт rune (Unicode code point).

```go
s := "привет"
len(s)          // 12 (байты, не символы)
utf8.RuneCountInString(s) // 6

// Итерация по символам
for i, r := range s {
    fmt.Printf("%d: %c\n", i, r)
}

// Эффективная конкатенация
var b strings.Builder
for _, word := range words {
    b.WriteString(word)
}
result := b.String()
```

## Типичные приёмы

### Разворот массива in-place

```go
func reverse(arr []int) {
    for i, j := 0, len(arr)-1; i < j; i, j = i+1, j-1 {
        arr[i], arr[j] = arr[j], arr[i]
    }
}
```

### Удаление дубликатов из отсортированного массива

```go
func removeDuplicates(nums []int) int {
    if len(nums) == 0 {
        return 0
    }
    slow := 0
    for fast := 1; fast < len(nums); fast++ {
        if nums[fast] != nums[slow] {
            slow++
            nums[slow] = nums[fast]
        }
    }
    return slow + 1
}
```

### Prefix Sum (кумулятивная сумма)

Быстрый ответ на запросы суммы подмассива.

```go
// Предвычисление: O(n)
prefix := make([]int, len(nums)+1)
for i, n := range nums {
    prefix[i+1] = prefix[i] + n
}

// Сумма на отрезке [l, r]: O(1)
sum := prefix[r+1] - prefix[l]
```

### Kadane's Algorithm (максимальная сумма подмассива)

```go
func maxSubarraySum(nums []int) int {
    maxSum, currSum := nums[0], nums[0]
    for _, n := range nums[1:] {
        currSum = max(n, currSum+n)
        maxSum = max(maxSum, currSum)
    }
    return maxSum
}
```
