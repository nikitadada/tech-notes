# Deployments

Декларативное управление ReplicaSet и Pod'ами. Deployment описывает желаемое состояние, а контроллер приводит кластер к нему.

## Что умеет

- Rolling update с нулевым downtime
- Откат к предыдущей версии (`rollout undo`)
- Масштабирование реплик
- Управление стратегией обновления

## Стратегии обновления

### RollingUpdate (по умолчанию)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # сколько Pod'ов сверх desired можно создать
    maxUnavailable: 0   # сколько Pod'ов может быть недоступно
```

- `maxSurge: 1, maxUnavailable: 0` — безопасный вариант, всегда есть нужное кол-во Pod'ов
- `maxSurge: 0, maxUnavailable: 1` — экономный, но на время обновления меньше Pod'ов

### Recreate

Убивает все старые Pod'ы, потом создаёт новые. Есть downtime. Используется, когда нельзя запустить две версии одновременно (миграции БД, конфликт портов).

## Пример манифеста

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: api-server:v1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: db-config
                  key: host
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
```

## Полезные команды

```bash
# Статус rollout
kubectl rollout status deployment/api-server

# История версий
kubectl rollout history deployment/api-server

# Откат к предыдущей версии
kubectl rollout undo deployment/api-server

# Откат к конкретной ревизии
kubectl rollout undo deployment/api-server --to-revision=2

# Масштабирование
kubectl scale deployment/api-server --replicas=5

# Обновление образа без правки манифеста
kubectl set image deployment/api-server api=api-server:v1.3.0
```

## Связь Deployment → ReplicaSet → Pod

```
Deployment (api-server)
  └── ReplicaSet (api-server-7d9f8b6c4)   ← текущая версия
  │     ├── Pod (api-server-7d9f8b6c4-abc)
  │     ├── Pod (api-server-7d9f8b6c4-def)
  │     └── Pod (api-server-7d9f8b6c4-ghi)
  └── ReplicaSet (api-server-5c8e7a3b1)   ← предыдущая версия (0 реплик)
```

При rolling update создаётся новый ReplicaSet, Pod'ы плавно переезжают.

## Best practices

- Всегда указывай `resources.requests` и `resources.limits`
- Используй `readinessProbe` — без него Pod получит трафик до готовности
- Указывай конкретный тег образа (`v1.2.0`), не `latest`
- Храни `revisionHistoryLimit` разумным (по умолчанию 10)
