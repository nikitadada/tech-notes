# Replication

Копирование данных на несколько узлов для отказоустойчивости, масштабирования чтения и снижения latency.

## Стратегии репликации

### Single-Leader (Master-Slave)

```
Client writes → Leader → Follower 1
                       → Follower 2
                       → Follower 3
Client reads  → любой узел
```

- Все записи через leader
- Чтение с любого узла (возможно stale)
- При отказе leader'а — failover

**Примеры:** PostgreSQL streaming replication, MySQL replication, Redis Sentinel.

**Проблемы:**
- Leader — single point of failure
- Replication lag при чтении с follower'ов

### Multi-Leader

```
Client A → Leader 1 ←→ Leader 2 ← Client B
              ↓            ↓
          Follower      Follower
```

- Несколько узлов принимают запись
- Запись реплицируется между leader'ами

**Применение:** multi-datacenter (leader в каждом DC), оффлайн-приложения.

**Проблемы:** конфликты при одновременной записи в разные leader'ы.

### Leaderless (Dynamo-style)

```
Client → Node 1 (write)
       → Node 2 (write)
       → Node 3 (write)
       
Client → Node 1 (read)
       → Node 2 (read)
       → выбрать самый свежий
```

- Запись и чтение на несколько узлов параллельно
- Кворум: W + R > N гарантирует чтение актуальных данных

**Примеры:** Cassandra, DynamoDB, Riak.

## Синхронная vs Асинхронная

### Синхронная репликация
- Запись подтверждена после записи на все реплики
- Сильная консистентность
- Выше latency, ниже availability

### Асинхронная репликация
- Запись подтверждена после записи на leader
- Ниже latency, выше availability
- Возможна потеря данных при отказе leader'а

### Полусинхронная (Semi-synchronous)
- Одна реплика синхронная, остальные асинхронные
- Компромисс: гарантия хотя бы одной актуальной копии

## Replication Lag

Задержка между записью на leader и её появлением на follower.

### Проблемы

**Read-after-write inconsistency:**
```
Client → Write(x=5) → Leader
Client → Read(x) → Follower → x=3 (ещё не реплицировано)
```

**Решения:**
- Read-your-writes: читай с leader'а после собственной записи
- Monotonic reads: закрепить клиента за одной репликой (session stickiness)
- Causal consistency: отслеживать зависимости между операциями

## Failover

При отказе leader'а:
1. Детектирование отказа (heartbeat timeout)
2. Выбор нового leader'а (наиболее актуальная реплика)
3. Переконфигурация (клиенты перенаправляются)

**Проблемы failover'а:**
- Асинхронный follower может потерять данные
- Split brain — два узла считают себя leader'ом
- Как определить что leader действительно мёртв (а не просто медленный)?

## Write-Ahead Log (WAL) Replication

Leader пишет WAL → отправляет WAL follower'ам → follower'ы применяют.

Используется в PostgreSQL (WAL shipping, streaming replication).

```
Leader WAL: [INSERT x=1] [UPDATE x=2] [DELETE y]
                 ↓            ↓           ↓
Follower WAL: [INSERT x=1] [UPDATE x=2] [DELETE y]
```
