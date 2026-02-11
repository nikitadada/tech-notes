# Raft

Алгоритм консенсуса для распределённых систем. Проще для понимания чем Paxos, широко используется (etcd, Consul, CockroachDB).

## Проблема

Несколько серверов должны согласованно реплицировать лог команд, даже если часть серверов недоступна.

## Роли

| Роль | Описание |
|------|----------|
| **Leader** | Принимает все запросы на запись, реплицирует лог |
| **Follower** | Пассивно реплицирует лог от leader'а |
| **Candidate** | Участвует в выборах нового leader'а |

В нормальном состоянии: 1 leader, N-1 followers.

## Три подзадачи

### 1. Leader Election

```
Follower → timeout (нет heartbeat) → Candidate → получил большинство → Leader
```

- Каждый узел имеет election timeout (рандомный, 150-300ms)
- Если follower не получает heartbeat — становится candidate
- Candidate увеличивает `term`, голосует за себя, запрашивает голоса
- Побеждает кандидат с большинством голосов
- Каждый узел голосует максимум за одного кандидата в term'е

**Split vote:** два candidate'а одновременно — ни один не набирает большинство → новый term, рандомный timeout решает.

### 2. Log Replication

```
Client → Leader → AppendEntries RPC → Followers
              ↓
        Commit (большинство подтвердило)
              ↓
        Apply to state machine
```

- Leader получает команду, добавляет в свой лог
- Отправляет `AppendEntries` всем follower'ам
- Когда большинство подтвердило — entry committed
- Committed entry применяется к state machine (необратимо)

### 3. Safety

**Election restriction:** кандидат может победить только если его лог не менее актуален чем у большинства.

**Leader Completeness:** committed entry гарантированно присутствует в логе каждого будущего leader'а.

## Term

Логический такт времени. Монотонно увеличивается.

```
Term 1: Leader=A  |  Term 2: Leader=B  |  Term 3: Leader=A
```

- Каждый term начинается с выборов
- Узел с меньшим term обновляет свой до актуального
- Запросы со старым term отклоняются

## Heartbeat

Leader периодически отправляет пустые `AppendEntries` (heartbeat) для:
- Подтверждения лидерства
- Предотвращения новых выборов

## Кворум

Для кластера из N узлов: кворум = ⌊N/2⌋ + 1

| Узлов | Кворум | Допустимых отказов |
|-------|--------|--------------------|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

## Пример: etcd

etcd — distributed KV store, использует Raft.

```bash
# Состояние кластера
etcdctl endpoint status --write-out=table

# Список членов
etcdctl member list --write-out=table
```

## Ограничения

- Один leader — bottleneck на запись
- Работает при отказе меньшинства узлов
- Не byzantine fault tolerant (доверенные узлы)
- Latency зависит от RTT до большинства
