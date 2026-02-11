# Hash Tables

Структура данных для хранения пар ключ-значение с O(1) средним временем доступа.

## Принцип работы

```
key → hash(key) → index → bucket → value
```

1. Хеш-функция преобразует ключ в целое число
2. Индекс = hash % capacity
3. Значение хранится в bucket по индексу

## Разрешение коллизий

### Chaining (цепочки)

Каждый bucket — связный список (или slice). При коллизии элемент добавляется в список.

```
Bucket 0: → (key1, val1) → (key5, val5)
Bucket 1: → (key2, val2)
Bucket 2: → пусто
Bucket 3: → (key3, val3) → (key7, val7) → (key9, val9)
```

### Open Addressing (открытая адресация)

При коллизии ищем следующий свободный bucket.

- **Linear probing:** index+1, index+2, ...
- **Quadratic probing:** index+1², index+2², ...
- **Double hashing:** index + i*hash2(key)

Go `map` использует chaining с buckets по 8 элементов.

## Map в Go

```go
m := make(map[string]int)
m["key"] = 42

// Проверка наличия
val, ok := m["key"]
if !ok {
    // ключ отсутствует
}

// Удаление
delete(m, "key")

// Итерация (порядок не гарантирован!)
for k, v := range m {
    fmt.Println(k, v)
}
```

### Внутреннее устройство Go map

- Массив buckets, каждый bucket хранит до 8 пар key-value
- При заполнении > 6.5 элементов/bucket → рехеширование (grow)
- Рехеширование происходит инкрементально (не блокирует)
- **Не потокобезопасен** — concurrent read/write → panic

### sync.Map

Потокобезопасный map для специфичных паттернов:

```go
var m sync.Map

m.Store("key", 42)
val, ok := m.Load("key")
m.Delete("key")

// Атомарная загрузка-или-сохранение
val, loaded := m.LoadOrStore("key", 42)
```

Эффективен когда:
- Ключи записываются один раз, читаются много
- Горутины работают с непересекающимися ключами

Для остальных случаев — `sync.RWMutex` + обычный map.

## Load Factor

```
load_factor = количество_элементов / количество_buckets
```

- Высокий LF → больше коллизий → медленнее
- Обычный порог для рехеширования: 0.75
- Go map: ~6.5 элементов на bucket

## Хеш-функции

| Функция | Описание |
|---------|----------|
| FNV | Простая, быстрая, используется в Go для map |
| MurmurHash | Хорошее распределение, не криптографическая |
| SHA-256 | Криптографическая, медленная для hash map |
| xxHash | Очень быстрая, хорошее распределение |

## Типичные задачи с hash map

### Подсчёт частоты

```go
freq := make(map[rune]int)
for _, ch := range s {
    freq[ch]++
}
```

### Группировка анаграмм

```go
func groupAnagrams(strs []string) [][]string {
    groups := make(map[string][]string)
    for _, s := range strs {
        sorted := sortString(s)
        groups[sorted] = append(groups[sorted], s)
    }
    result := make([][]string, 0, len(groups))
    for _, group := range groups {
        result = append(result, group)
    }
    return result
}
```

### Set (множество)

```go
type Set[T comparable] map[T]struct{}

func (s Set[T]) Add(val T)         { s[val] = struct{}{} }
func (s Set[T]) Contains(val T) bool { _, ok := s[val]; return ok }
func (s Set[T]) Remove(val T)      { delete(s, val) }
```
