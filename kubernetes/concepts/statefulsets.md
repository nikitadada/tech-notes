# StatefulSets

Контроллер для stateful-приложений, которым нужны стабильные сетевые идентификаторы и персистентное хранилище.

## Отличия от Deployment

| Свойство | Deployment | StatefulSet |
|----------|-----------|-------------|
| Имена Pod'ов | Случайные (`app-7d9f8-abc`) | Порядковые (`app-0`, `app-1`, `app-2`) |
| Порядок создания | Параллельно | Последовательно (0 → 1 → 2) |
| Порядок удаления | Любой | Обратный (2 → 1 → 0) |
| Хранилище | Общий PVC или без | Индивидуальный PVC для каждого Pod |
| DNS-имя | Нет стабильного | `<pod-name>.<service-name>.<namespace>.svc.cluster.local` |

## Когда использовать

- Базы данных (PostgreSQL, MySQL, MongoDB)
- Message brokers (Kafka, RabbitMQ)
- Distributed caches (Redis Cluster, Elasticsearch)
- Любое приложение, где важен порядок и идентичность узлов

## Пример манифеста

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pg-secret
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

## Headless Service

StatefulSet требует Headless Service (`clusterIP: None`) для создания DNS-записей:

```
postgres-0.postgres-headless.default.svc.cluster.local
postgres-1.postgres-headless.default.svc.cluster.local
postgres-2.postgres-headless.default.svc.cluster.local
```

Это позволяет обращаться к конкретному Pod'у по имени.

## Update strategies

- **RollingUpdate** (default) — обновляет Pod'ы в обратном порядке (N-1 → 0)
- **OnDelete** — обновляет только при ручном удалении Pod'а
- `partition` — обновляет только Pod'ы с ordinal >= partition

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2  # обновит только pod-2, pod-0 и pod-1 останутся на старой версии
```

## Важно помнить

- PVC не удаляются при удалении StatefulSet — данные сохраняются
- При уменьшении реплик PVC остаются, при увеличении — переиспользуются
- Не подходит для stateless-приложений — используй Deployment
