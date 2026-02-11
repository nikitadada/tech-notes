# Paxos

Фундаментальный алгоритм консенсуса (Lamport, 1998). Теоретически элегантен, но сложен в реализации. Основа для многих production-систем (Chubby, Spanner).

## Задача

Набор узлов должен согласиться на одно значение (consensus), даже при отказах части узлов.

## Роли

| Роль | Описание |
|------|----------|
| **Proposer** | Предлагает значение |
| **Acceptor** | Голосует за/против предложения |
| **Learner** | Узнаёт выбранное значение |

Один узел может выполнять несколько ролей одновременно.

## Basic Paxos (Single-Decree)

Две фазы для согласования одного значения.

### Phase 1: Prepare

```
Proposer → Prepare(n) → Acceptors
         ← Promise(n, accepted_value?) ←
```

1. Proposer выбирает уникальный номер `n` (proposal number)
2. Отправляет `Prepare(n)` acceptor'ам
3. Acceptor: если `n` > всех ранее виденных номеров → обещает не принимать предложения с номером < n, возвращает ранее принятое значение (если есть)

### Phase 2: Accept

```
Proposer → Accept(n, value) → Acceptors
         ← Accepted(n) ←
```

1. Если proposer получил Promise от большинства:
   - Если кто-то вернул accepted value → использует его (нельзя выбрать своё)
   - Иначе → предлагает своё значение
2. Отправляет `Accept(n, value)` acceptor'ам
3. Acceptor: принимает, если не обещал номер > n

**Значение выбрано** когда большинство acceptor'ов его приняли.

## Multi-Paxos

Basic Paxos — один раунд для одного значения. Multi-Paxos оптимизирует для последовательности значений (replicated log).

**Оптимизация:** стабильный leader пропускает Phase 1 для последующих слотов.

```
Slot 1: Full Paxos (Phase 1 + Phase 2) → Leader elected
Slot 2: Phase 2 only (leader already established)
Slot 3: Phase 2 only
...
```

## Paxos vs Raft

| Аспект | Paxos | Raft |
|--------|-------|------|
| Понятность | Сложный | Простой (by design) |
| Leader | Опциональный (Multi-Paxos) | Обязательный |
| Log gaps | Возможны | Нет (лог непрерывный) |
| Membership change | Сложно | Встроено |
| Реализации | Chubby, Spanner | etcd, Consul, CockroachDB |

## Варианты Paxos

- **Fast Paxos** — 2 фазы вместо 3 в оптимистичном случае
- **Cheap Paxos** — экономия на acceptor'ах (f+1 активных, f запасных)
- **EPaxos (Egalitarian Paxos)** — без выделенного leader'а, параллельные commit'ы
- **Flexible Paxos** — разные размеры кворумов для Phase 1 и Phase 2

## FLP Impossibility

Невозможность гарантировать consensus в асинхронной системе с хотя бы одним отказом (Fischer, Lynch, Paterson, 1985).

Paxos обходит это через рандомизацию (случайные таймауты) — гарантирует safety всегда, liveness — в конечном итоге.
