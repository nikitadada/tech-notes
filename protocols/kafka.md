# Apache Kafka

Распределённая платформа потоковой обработки событий. Высокопроизводительный, персистентный message broker.

## Основные концепции

### Topic
Именованный лог сообщений. Аналогия: таблица в БД.

### Partition
Topic делится на partition'ы — параллельные log-файлы. Сообщения внутри partition строго упорядочены.

### Offset
Порядковый номер сообщения в partition. Уникален в пределах partition.

### Consumer Group
Группа консьюмеров, каждый из которых читает уникальный набор partition'ов одного topic'а.

```
Topic (3 partitions) → Consumer Group (3 consumers)
  Partition 0 → Consumer A
  Partition 1 → Consumer B
  Partition 2 → Consumer C
```

Если консьюмеров больше чем partition'ов — лишние простаивают.

## Архитектура

```
Producer → Broker Cluster → Consumer
              │
        ┌─────┼─────┐
        │     │     │
     Broker  Broker  Broker
     P0,P1   P2,P3  P0(r),P2(r)
```

- **Broker** — сервер Kafka, хранит partition'ы
- **Replication** — каждый partition реплицируется на несколько брокеров
- **Leader/Follower** — запись/чтение через leader, followers реплицируют
- **ISR (In-Sync Replicas)** — реплики, которые не отстают от leader'а

## Гарантии доставки

| Настройка `acks` | Описание | Надёжность |
|-------------------|----------|-----------|
| `acks=0` | Не ждёт подтверждения | Потеря возможна |
| `acks=1` | Подтверждение от leader'а | Потеря при падении leader'а |
| `acks=all` | Подтверждение от всех ISR | Максимальная |

### Семантики доставки

- **At most once** — сообщение может быть потеряно (commit offset до обработки)
- **At least once** — сообщение может быть обработано дважды (commit offset после обработки)
- **Exactly once** — через idempotent producer + transactional API

## Конфигурация

### Producer

```properties
acks=all
retries=3
enable.idempotence=true
max.in.flight.requests.per.connection=5
compression.type=snappy
batch.size=16384
linger.ms=5
```

### Consumer

```properties
group.id=order-service
auto.offset.reset=earliest    # или latest
enable.auto.commit=false      # ручной commit для at-least-once
max.poll.records=500
session.timeout.ms=30000
```

## Паттерны

### Event Sourcing
Каждое изменение — event в Kafka. Состояние восстанавливается из replay событий.

### CQRS
Запись → Kafka → Read model обновляется асинхронно.

### Dead Letter Queue
Невалидные сообщения перенаправляются в отдельный topic для анализа.

```
main-topic → consumer → [ошибка] → dlq-topic
```

### Compacted Topics
Kafka хранит только последнее значение для каждого ключа. Используется для CDC, snapshot'ов состояния.

```properties
cleanup.policy=compact
```

## Мониторинг

Ключевые метрики:
- **Consumer lag** — отставание consumer'а от последнего offset'а
- **Under-replicated partitions** — partition'ы с ISR < replication factor
- **Request latency** — время обработки запросов брокером
- **Disk usage** — заполненность хранилища

```bash
# Consumer lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-service --describe
```

## Kafka vs RabbitMQ

| Критерий | Kafka | RabbitMQ |
|----------|-------|----------|
| Модель | Log (pull) | Queue (push) |
| Порядок | В пределах partition | В пределах queue |
| Хранение | Персистентное (retention) | До consume |
| Throughput | Очень высокий | Средний |
| Replay | Да | Нет |
| Роутинг | По partition key | Exchanges, bindings |
| Применение | Event streaming, CDC | Task queue, RPC |
