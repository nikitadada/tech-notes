# Leader Election

Выбор одного узла-координатора в распределённой системе. Только leader выполняет определённые задачи (запись, координация).

## Зачем нужен leader

- Координация записи (single-leader replication)
- Распределение задач (scheduler)
- Управление distributed lock
- Избежание конфликтов при concurrent доступе

## Подходы

### 1. Через distributed lock

```
Node A → tryLock("/leader") → acquired → я leader
Node B → tryLock("/leader") → failed → я follower
Node C → tryLock("/leader") → failed → я follower
```

Leader периодически продлевает lock (TTL). Если не продлил — другой узел захватывает.

**Инструменты:** etcd, ZooKeeper, Consul.

### 2. Через consensus (Raft/Paxos)

Встроен в алгоритм. Leader выбирается через голосование (см. [Raft](../protocols/consensus/raft.md)).

### 3. Bully Algorithm

Узел с наибольшим ID становится leader'ом.

```
1. Узел обнаруживает, что leader мёртв
2. Отправляет Election message всем узлам с бо́льшим ID
3. Если никто не ответил — становится leader'ом
4. Если ответили — ждёт, пока тот станет leader'ом
```

Простой, но не устойчив к частым отказам.

## Leader Election через etcd

```go
import (
    clientv3 "go.etcd.io/etcd/client/v3"
    "go.etcd.io/etcd/client/v3/concurrency"
)

func runLeaderElection(ctx context.Context, client *clientv3.Client, nodeID string) error {
    session, err := concurrency.NewSession(client, concurrency.WithTTL(10))
    if err != nil {
        return err
    }
    defer session.Close()

    election := concurrency.NewElection(session, "/service/leader")

    // Блокируется до избрания
    if err := election.Campaign(ctx, nodeID); err != nil {
        return err
    }

    log.Printf("node %s elected as leader", nodeID)

    // Работа в роли leader'а
    doLeaderWork(ctx)

    // Добровольная отставка
    return election.Resign(ctx)
}
```

## Leader Election через Kubernetes

### Lease API

```go
import (
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
)

lock := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{
        Name:      "my-service-leader",
        Namespace: "default",
    },
    Client: client.CoordinationV1(),
    LockConfig: resourcelock.ResourceLockConfig{
        Identity: hostname,
    },
}

leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock:          lock,
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            log.Println("became leader")
            runLeader(ctx)
        },
        OnStoppedLeading: func() {
            log.Println("lost leadership")
        },
    },
})
```

## Проблемы

### Split Brain

Два узла считают себя leader'ом. Причины:
- Network partition
- GC pause / stop-the-world
- Lock TTL истёк, но старый leader ещё работает

**Решение: Fencing token**

```
Leader A → получает lock с token=33
Leader A → GC pause (lock expires)
Leader B → получает lock с token=34
Leader A → просыпается, пытается писать с token=33
Storage  → отклоняет token=33 (видел 34)
```

### Thundering Herd

Leader падает → все followers одновременно пытаются стать leader'ом.

**Решение:** Jitter — случайная задержка перед попыткой захвата lock.

## Когда НЕ нужен leader

- Leaderless системы (Cassandra, DynamoDB)
- CRDTs — конвергенция без координации
- Если можно допустить дублирование работы (idempotent operations)
