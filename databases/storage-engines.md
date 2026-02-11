# Storage Engines

Движки хранения определяют как данные записываются на диск и читаются.

## B-Tree based (PostgreSQL, MySQL InnoDB)

Классическая модель. Данные хранятся в страницах B-дерева.

### Как работает запись

```
1. Запись в WAL (Write-Ahead Log) → flush на диск
2. Обновление страницы в buffer pool (RAM)
3. Background writer сбрасывает грязные страницы на диск
```

**WAL** гарантирует durability: при сбое данные восстанавливаются из лога.

### Плюсы / Минусы

- ✅ Хорошая производительность чтения
- ✅ Поддержка range queries
- ✅ Зрелая технология, предсказуемая производительность
- ❌ Write amplification (обновление страницы → перезапись всей страницы)
- ❌ Рандомный I/O при записи

## LSM-Tree (Log-Structured Merge Tree)

Используется в RocksDB, LevelDB, Cassandra, ScyllaDB.

### Как работает запись

```
1. Запись в WAL
2. Запись в MemTable (in-memory sorted structure)
3. Когда MemTable заполнен → flush в SSTable на диск (Sorted String Table)
4. Background compaction: merge SSTables → уменьшение количества файлов
```

### Как работает чтение

```
1. Проверить MemTable
2. Проверить Bloom filter каждого SSTable
3. Читать подходящие SSTables (от новых к старым)
```

### Compaction strategies

| Стратегия | Описание |
|-----------|----------|
| Size-tiered | SSTables объединяются при достижении порога. Проще, больше space amplification |
| Leveled | SSTables организованы в уровни (L0, L1, ...). Лучше read, больше write amplification |
| FIFO | Старые SSTables удаляются. Для time-series данных |

### Плюсы / Минусы

- ✅ Высокая пропускная способность записи (sequential I/O)
- ✅ Хорошее сжатие данных
- ❌ Read amplification (проверка нескольких SSTables)
- ❌ Space amplification (данные дублируются до compaction)
- ❌ Compaction потребляет CPU и I/O

## Сравнение

| Свойство | B-Tree | LSM-Tree |
|----------|--------|----------|
| Чтение (point) | Быстрое | Медленнее (несколько файлов) |
| Чтение (range) | Быстрое | Медленнее |
| Запись | Средняя | Быстрая |
| Write amplification | Высокая | Средняя |
| Space amplification | Низкая | Средняя-высокая |
| Применение | OLTP, mixed workload | Write-heavy, time-series |

## Column-oriented storage

Данные хранятся по колонкам, а не по строкам. Для аналитики (OLAP).

```
Row-oriented:     Column-oriented:
[id, name, age]   id:   [1, 2, 3, 4]
[1, Alice, 25]    name: [Alice, Bob, Carol, Dave]
[2, Bob, 30]      age:  [25, 30, 28, 35]
[3, Carol, 28]
[4, Dave, 35]
```

**Преимущества:**
- Чтение только нужных колонок (меньше I/O)
- Отличное сжатие (одинаковые типы данных рядом)
- Vectorized execution

**Системы:** ClickHouse, Apache Parquet, Amazon Redshift.

## Page structure (PostgreSQL)

```
Page (8KB default):
┌────────────────────────┐
│ Page Header (24 bytes) │
├────────────────────────┤
│ Item Pointers          │ → указатели на tuples
├────────────────────────┤
│ Free Space             │
├────────────────────────┤
│ Tuples (row data)      │ ← данные строк
├────────────────────────┤
│ Special Space          │
└────────────────────────┘
```

## Buffer Pool

Кэш страниц в RAM. Основной механизм производительности.

```
Query → Buffer Pool (RAM) → hit → return
                           → miss → read from disk → cache → return
```

- `shared_buffers` в PostgreSQL (обычно 25% RAM)
- LRU или clock-sweep для вытеснения
- Dirty pages периодически записываются на диск (checkpoint)
