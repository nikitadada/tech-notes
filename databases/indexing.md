# Indexing (Индексы)

Структуры данных для ускорения поиска в БД. Без индекса — full table scan (O(n)).

## B-Tree (B+ Tree)

Стандартный индекс в большинстве РСУБД. Сбалансированное дерево, где каждый узел — страница на диске.

```
              [10 | 20 | 30]              ← корень
             /    |     |    \
      [1-9]   [11-19] [21-29] [31-40]    ← внутренние узлы
       / \      / \      / \     / \
    [data] [data] [data] [data] [data]    ← листья (указатели на строки)
```

**Свойства:**
- Упорядочен → поддерживает range queries, ORDER BY
- O(log n) поиск, вставка, удаление
- Хорошо работает с disk I/O (один узел = одна страница)

**PostgreSQL:**

```sql
-- Создание B-Tree индекса (по умолчанию)
CREATE INDEX idx_users_email ON users(email);

-- Составной индекс
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Уникальный
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
```

## Hash Index

Hash-таблица. Только exact match (=), не поддерживает range queries.

```sql
CREATE INDEX idx_users_email_hash ON users USING hash(email);
```

O(1) поиск, но:
- Нет range queries, сортировки
- Не поддерживает partial matching

## GIN (Generalized Inverted Index)

Для полнотекстового поиска, JSONB, массивов.

```sql
-- Полнотекстовый поиск
CREATE INDEX idx_articles_body ON articles USING gin(to_tsvector('russian', body));

-- JSONB
CREATE INDEX idx_data_jsonb ON events USING gin(data jsonb_path_ops);

-- Массивы
CREATE INDEX idx_tags ON posts USING gin(tags);
```

## GiST (Generalized Search Tree)

Для геоданных, диапазонов, полнотекстового поиска.

```sql
-- Геоиндекс
CREATE INDEX idx_locations_point ON locations USING gist(coordinates);

-- Range types
CREATE INDEX idx_reservations ON reservations USING gist(during);
```

## Covering Index (INCLUDE)

Индекс содержит дополнительные колонки, что позволяет выполнить запрос без обращения к таблице (Index-Only Scan).

```sql
CREATE INDEX idx_orders_covering ON orders(user_id)
  INCLUDE (total, status);

-- Запрос использует только индекс:
SELECT total, status FROM orders WHERE user_id = 123;
```

## Partial Index

Индекс только для подмножества строк. Экономит место, быстрее обновляется.

```sql
-- Индекс только для активных пользователей
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Индекс только для необработанных заказов
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'pending';
```

## Составной индекс (Composite)

Порядок колонок важен! Работает правило leftmost prefix.

```sql
CREATE INDEX idx_abc ON table(a, b, c);

-- Использует индекс:
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3

-- НЕ использует индекс:
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

## EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE user_id = 123 AND status = 'pending';

-- Что смотреть:
-- Seq Scan → нет подходящего индекса
-- Index Scan → индекс используется
-- Index Only Scan → все данные в индексе (covering)
-- Bitmap Index Scan → индекс для фильтрации, потом heap access
-- actual time → реальное время выполнения
-- rows → количество обработанных строк
```

## Антипаттерны

- Индекс на каждую колонку — замедляет запись
- Неиспользуемые индексы — занимают место, тормозят INSERT/UPDATE
- Индекс на low-cardinality колонку (boolean, status с 3 значениями) — бесполезен для B-Tree
- Функция в WHERE без функционального индекса: `WHERE LOWER(email) = ...`

```sql
-- Проверка неиспользуемых индексов (PostgreSQL)
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0;
```
