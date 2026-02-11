# Sharding (Партиционирование)

Разделение данных на части (shards/partitions), каждая хранится на отдельном узле. Масштабирование записи и хранения горизонтально.

## Зачем

- Данные не помещаются на один узел
- Нужна горизонтальная масштабируемость записи
- Распределение нагрузки между серверами

## Стратегии шардирования

### Key-based (Hash)

```
shard = hash(key) % num_shards
```

- Равномерное распределение данных
- Нет range queries
- Рехеширование при добавлении шардов — необходим consistent hashing

**Пример:** `hash(user_id) % 4` → shard 0-3

### Range-based

```
Shard 0: A-G
Shard 1: H-N
Shard 2: O-U
Shard 3: V-Z
```

- Поддерживает range queries
- Возможен hot spot (неравномерное распределение)
- Нужна ребалансировка при skew

### Directory-based

Lookup service хранит маппинг key → shard.

- Гибкость в распределении
- Lookup service — single point of failure
- Дополнительный hop на каждый запрос

## Consistent Hashing

Решает проблему рехеширования при добавлении/удалении узлов.

```
         Node A
        /      \
    Key1        Key2
   /                \
Node D              Node B
   \                /
    Key4        Key3
        \      /
         Node C
```

- Ключи и узлы хешируются на кольцо
- Ключ принадлежит первому узлу по часовой стрелке
- При добавлении узла перераспределяется только часть ключей
- Virtual nodes — каждый физический узел имеет несколько позиций на кольце (для равномерности)

## Cross-shard операции

Главная сложность шардирования.

### Cross-shard joins
Невозможны на уровне БД → денормализация или application-level joins.

### Distributed transactions
2PC (Two-Phase Commit) — медленно, блокирует при отказах.

```
Coordinator → Prepare → Shard A: OK
                      → Shard B: OK
Coordinator → Commit  → Shard A: committed
                      → Shard B: committed
```

### Saga pattern
Последовательность локальных транзакций с компенсирующими действиями при ошибке.

```
1. Debit account A (compensate: credit A)
2. Credit account B (compensate: debit B)
3. Send notification
```

## Rebalancing

Перераспределение данных при добавлении/удалении шардов.

**Подходы:**
- **Fixed partitions** — создать больше partition'ов, чем узлов. При добавлении узла — перенести partition'ы
- **Dynamic splitting** — partition делится при превышении размера
- **Proportional** — количество partition'ов пропорционально данным на узле

## Шардирование в разных системах

| Система | Стратегия | Ключ шардирования |
|---------|-----------|-------------------|
| PostgreSQL (Citus) | Hash / Range | Указывается при создании таблицы |
| MongoDB | Hash / Range | Shard key |
| Cassandra | Consistent Hash | Partition key |
| Vitess (MySQL) | Range / Hash | Vindex |
| Redis Cluster | Hash slots (16384) | CRC16(key) % 16384 |

## Антипаттерны

- **Hot shard** — один шард получает непропорционально много запросов. Решение: compound shard key, salting
- **Cross-shard queries** — запрос ко всем шардам. Решение: денормализация, правильный выбор shard key
- **Слишком мало шардов** — невозможно масштабировать дальше
