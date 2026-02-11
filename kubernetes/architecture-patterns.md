# Architecture Patterns в Kubernetes

## Sidecar Pattern

Вспомогательный контейнер рядом с основным в одном Pod'е.

**Применения:**
- Логирование (filebeat, fluentd)
- Прокси (envoy, nginx)
- Синхронизация конфигов
- TLS termination

```yaml
spec:
  containers:
    - name: app
      image: app:v1
      ports:
        - containerPort: 8080
    - name: log-shipper
      image: fluentd:v1.16
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

## Ambassador Pattern

Sidecar-контейнер выступает прокси к внешним сервисам. Основное приложение обращается к localhost, ambassador маршрутизирует наружу.

Пример: прокси к разным инстансам Redis (master/slave) в зависимости от типа запроса.

## Init Container Pattern

Контейнер, который выполняется до старта основных. Полезен для:
- Ожидания зависимостей
- Миграций БД
- Скачивания конфигов/секретов
- Настройки permissions

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox
      command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
    - name: migrate
      image: app:v1
      command: ['./migrate', 'up']
  containers:
    - name: app
      image: app:v1
```

## Blue-Green Deployment

Два идентичных окружения. Трафик переключается целиком.

```bash
# Текущий (blue) Service указывает на blue Pod'ы
# Деплоим green версию
kubectl apply -f deployment-green.yaml

# Проверяем green
kubectl port-forward svc/green-service 8080:80

# Переключаем трафик
kubectl patch service app-service -p '{"spec":{"selector":{"version":"green"}}}'

# Откат — переключить selector обратно на blue
```

## Canary Deployment

Постепенное переключение трафика на новую версию.

```yaml
# Canary deployment (10% трафика)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-canary
spec:
  replicas: 1           # 1 из 10 общих реплик
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app      # тот же label что у stable
        version: canary
    spec:
      containers:
        - name: app
          image: app:v2  # новая версия
```

Service с selector `app: my-app` балансирует между stable (9 реплик) и canary (1 реплика).

Для точного управления трафиком — Istio VirtualService или Argo Rollouts.

## Namespace Isolation

Разделение окружений через namespaces + NetworkPolicy + ResourceQuota.

```yaml
# ResourceQuota для namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    pods: "50"
```

## GitOps

Декларативное управление кластером через Git-репозиторий.

**Инструменты:**
- **ArgoCD** — UI, Application CRD, auto-sync
- **Flux** — lightweight, Git-native
- **Helmfile** — управление Helm-чартами

Принцип: Git — single source of truth. Изменения через PR, автоматический sync в кластер.
