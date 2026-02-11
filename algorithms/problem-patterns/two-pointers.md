# Two Pointers

Два указателя двигаются по массиву/строке, сужая пространство поиска. Обычно превращает O(n²) в O(n).

## Варианты

### 1. Навстречу друг другу (opposing)

Один указатель в начале, другой в конце. Двигаются навстречу.

```
[1, 2, 3, 4, 5, 6, 7]
 ^                    ^
 left              right
```

**Пример: Two Sum в отсортированном массиве**

```go
func twoSum(nums []int, target int) []int {
    left, right := 0, len(nums)-1
    for left < right {
        sum := nums[left] + nums[right]
        switch {
        case sum == target:
            return []int{left, right}
        case sum < target:
            left++
        default:
            right--
        }
    }
    return nil
}
```

**Пример: Container With Most Water**

```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    best := 0
    for left < right {
        h := min(height[left], height[right])
        area := h * (right - left)
        best = max(best, area)
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }
    return best
}
```

### 2. В одном направлении (fast/slow)

Оба указателя идут слева направо, но с разной скоростью.

**Пример: Remove Duplicates from Sorted Array**

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

**Пример: Move Zeroes**

```go
func moveZeroes(nums []int) {
    slow := 0
    for fast := 0; fast < len(nums); fast++ {
        if nums[fast] != 0 {
            nums[slow], nums[fast] = nums[fast], nums[slow]
            slow++
        }
    }
}
```

### 3. Three Sum (расширение two pointers)

Фиксируй один элемент, для остальных используй two pointers.

```go
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    var result [][]int

    for i := 0; i < len(nums)-2; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue // пропуск дубликатов
        }
        left, right := i+1, len(nums)-1
        for left < right {
            sum := nums[i] + nums[left] + nums[right]
            switch {
            case sum == 0:
                result = append(result, []int{nums[i], nums[left], nums[right]})
                for left < right && nums[left] == nums[left+1] { left++ }
                for left < right && nums[right] == nums[right-1] { right-- }
                left++
                right--
            case sum < 0:
                left++
            default:
                right--
            }
        }
    }
    return result
}
```

## Когда применять

- Отсортированный массив + поиск пары с условием → opposing pointers
- Фильтрация/перестановка in-place → fast/slow
- Палиндром → opposing pointers
- Linked list: цикл, середина → fast/slow
