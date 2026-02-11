# Database Design

## Нормализация

Устранение избыточности данных через разделение на связанные таблицы.

### Нормальные формы

| НФ | Правило |
|----|---------|
| 1НФ | Атомарные значения, нет повторяющихся групп |
| 2НФ | 1НФ + нет частичных зависимостей от составного ключа |
| 3НФ | 2НФ + нет транзитивных зависимостей |

**Пример нарушения 3НФ:**

```
orders(id, customer_id, customer_name, customer_email)
```

`customer_name` и `customer_email` зависят от `customer_id`, а не от `id`. Решение: вынести в таблицу `customers`.

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    total NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

## Денормализация

Намеренное дублирование данных для ускорения чтения. Используй когда:
- JOIN'ы тормозят
- Данные читаются гораздо чаще, чем записываются
- Нужна высокая скорость read-heavy запросов

```sql
-- Денормализованная таблица для быстрых листингов
CREATE TABLE order_listings (
    order_id BIGINT,
    customer_name TEXT,       -- дублирование из customers
    total NUMERIC(10, 2),
    status TEXT,
    created_at TIMESTAMPTZ
);
```

Обновление через триггеры или асинхронную синхронизацию.

## Типы связей

```sql
-- One-to-Many: пользователь → заказы
CREATE TABLE orders (
    user_id BIGINT REFERENCES users(id)
);

-- Many-to-Many: пользователи ↔ роли
CREATE TABLE user_roles (
    user_id BIGINT REFERENCES users(id),
    role_id BIGINT REFERENCES roles(id),
    PRIMARY KEY (user_id, role_id)
);

-- One-to-One: пользователь → профиль
CREATE TABLE profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id)
);
```

## Выбор типов данных

| Данные | Тип PostgreSQL | Заметки |
|--------|---------------|---------|
| ID | `BIGSERIAL` / `UUID` | BIGSERIAL для внутренних, UUID для публичных |
| Деньги | `NUMERIC(12,2)` | Никогда `FLOAT` — потеря точности |
| Время | `TIMESTAMPTZ` | Всегда с timezone |
| Статус | `TEXT` + CHECK | Или отдельная таблица для расширяемости |
| JSON | `JSONB` | Не `JSON` — jsonb эффективнее для запросов |
| IP-адрес | `INET` | Встроенный тип PostgreSQL |

## Миграции

Принципы безопасных миграций (zero-downtime):

### ✅ Безопасно

```sql
-- Добавить nullable колонку
ALTER TABLE users ADD COLUMN phone TEXT;

-- Добавить индекс конкурентно
CREATE INDEX CONCURRENTLY idx_users_phone ON users(phone);

-- Добавить новую таблицу
CREATE TABLE audit_log (...);
```

### ❌ Опасно (блокирует таблицу)

```sql
-- Добавить NOT NULL колонку без default
ALTER TABLE users ADD COLUMN phone TEXT NOT NULL;

-- Решение: добавить nullable → заполнить → добавить constraint
ALTER TABLE users ADD COLUMN phone TEXT;
UPDATE users SET phone = '' WHERE phone IS NULL;
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- Добавить индекс без CONCURRENTLY
CREATE INDEX idx_users_phone ON users(phone);  -- блокирует запись
```

## Паттерны проектирования

### Soft Delete

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
CREATE INDEX idx_users_active ON users(id) WHERE deleted_at IS NULL;

-- "Удаление"
UPDATE users SET deleted_at = now() WHERE id = 123;

-- Запросы всегда фильтруют
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Audit Log

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name TEXT NOT NULL,
    record_id BIGINT NOT NULL,
    action TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    old_data JSONB,
    new_data JSONB,
    changed_by BIGINT,
    changed_at TIMESTAMPTZ DEFAULT now()
);
```

### Polymorphic associations (через JSONB)

```sql
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_events_type ON events(event_type, created_at);
CREATE INDEX idx_events_payload ON events USING gin(payload jsonb_path_ops);
```

## Checklist нового проекта

- [ ] Выбрать стратегию ID (BIGSERIAL vs UUID)
- [ ] TIMESTAMPTZ для всех временных колонок
- [ ] Foreign keys для referential integrity
- [ ] Индексы на колонки в WHERE и JOIN
- [ ] NOT NULL по умолчанию, nullable — осознанный выбор
- [ ] Миграции через версионированные файлы (goose, flyway)
- [ ] Мониторинг slow queries (`pg_stat_statements`)
