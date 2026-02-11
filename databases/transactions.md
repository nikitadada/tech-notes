# Transactions (Транзакции)

Группа операций, выполняемых атомарно. Либо все операции применяются, либо ни одна.

## ACID

| Свойство | Описание |
|----------|----------|
| **Atomicity** | Транзакция выполняется целиком или откатывается |
| **Consistency** | БД переходит из одного валидного состояния в другое |
| **Isolation** | Параллельные транзакции не влияют друг на друга |
| **Durability** | Закоммиченные данные сохраняются даже при сбое |

## Уровни изоляции

### Read Uncommitted
Видны незакоммиченные данные (dirty reads). Практически не используется.

### Read Committed (PostgreSQL default)
Видны только закоммиченные данные. Но повторное чтение может дать другой результат.

```
T1: SELECT balance → 100
T2: UPDATE balance = 50; COMMIT
T1: SELECT balance → 50  ← non-repeatable read
```

### Repeatable Read
Данные, прочитанные в начале транзакции, не меняются. Но могут появиться новые строки (phantom reads).

```
T1: SELECT COUNT(*) WHERE status='active' → 10
T2: INSERT INTO ... (status='active'); COMMIT
T1: SELECT COUNT(*) WHERE status='active' → 11  ← phantom read
```

В PostgreSQL Repeatable Read защищает и от phantom reads (через MVCC).

### Serializable
Полная изоляция. Результат как при последовательном выполнении. Самый медленный уровень.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

## Аномалии

| Аномалия | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|----------|:---:|:---:|:---:|:---:|
| Dirty read | Да | Нет | Нет | Нет |
| Non-repeatable read | Да | Да | Нет | Нет |
| Phantom read | Да | Да | Да* | Нет |
| Serialization anomaly | Да | Да | Да | Нет |

*В PostgreSQL RR защищает от phantom reads

## MVCC (Multi-Version Concurrency Control)

PostgreSQL не блокирует чтение при записи. Каждая строка имеет версии.

```
Row versions:
  xmin=100, xmax=∞    → версия 1 (создана транзакцией 100)
  xmin=105, xmax=∞    → версия 2 (создана транзакцией 105)
```

Каждая транзакция видит snapshot БД на момент своего начала.

**VACUUM** — очистка старых версий строк, которые никто не видит.

## Блокировки

### Row-level locks

```sql
-- Эксклюзивная блокировка строки
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- Shared lock (можно читать, нельзя писать)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- Пропустить заблокированные строки
SELECT * FROM tasks WHERE status = 'new'
  FOR UPDATE SKIP LOCKED
  LIMIT 1;
```

### Advisory locks

Пользовательские блокировки, не привязанные к строкам.

```sql
-- Захватить lock
SELECT pg_advisory_lock(hashtext('process_orders'));

-- Попытка (неблокирующая)
SELECT pg_try_advisory_lock(42);

-- Освободить
SELECT pg_advisory_unlock(hashtext('process_orders'));
```

## Deadlock

```
T1: LOCK row A → ждёт row B
T2: LOCK row B → ждёт row A
→ deadlock!
```

PostgreSQL автоматически обнаруживает deadlock и откатывает одну транзакцию.

**Предотвращение:** всегда блокируй ресурсы в одинаковом порядке.

## Паттерны

### Optimistic Locking

Не блокируем строку, проверяем при обновлении.

```sql
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 123 AND version = 5;

-- Если rows affected = 0 → конфликт, retry
```

### SELECT FOR UPDATE SKIP LOCKED (очередь задач)

```sql
BEGIN;
SELECT id, payload FROM tasks
  WHERE status = 'pending'
  FOR UPDATE SKIP LOCKED
  LIMIT 1;

-- обработка задачи

UPDATE tasks SET status = 'done' WHERE id = ?;
COMMIT;
```

Несколько воркеров могут параллельно разбирать задачи без конфликтов.
