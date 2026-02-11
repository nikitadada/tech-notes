# Pods

Минимальная деплоируемая единица в Kubernetes. Pod — это группа из одного или нескольких контейнеров с общими ресурсами (сеть, хранилище).

## Ключевые свойства

- Все контейнеры в Pod делят один IP-адрес и network namespace
- Контейнеры внутри Pod общаются через `localhost`
- Pod эфемерен — при падении создаётся новый (не перезапускается старый)
- Каждый Pod получает уникальный IP в кластере

## Жизненный цикл

```
Pending → Running → Succeeded/Failed
```

| Фаза | Описание |
|------|----------|
| `Pending` | Pod принят кластером, но контейнеры ещё не запущены |
| `Running` | Хотя бы один контейнер работает |
| `Succeeded` | Все контейнеры завершились успешно (exit 0) |
| `Failed` | Хотя бы один контейнер завершился с ошибкой |

## Probes

| Probe | Назначение |
|-------|-----------|
| `livenessProbe` | Перезапуск контейнера, если он завис |
| `readinessProbe` | Убрать Pod из Service, если не готов принимать трафик |
| `startupProbe` | Дать время на старт (отключает liveness/readiness до успеха) |

## Пример манифеста

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "64Mi"
          cpu: "100m"
        limits:
          memory: "128Mi"
          cpu: "250m"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 3
        periodSeconds: 5
```

## Multi-container patterns

- **Sidecar** — дополнительный контейнер для логирования, прокси (envoy, filebeat)
- **Init container** — запускается до основных, используется для миграций, ожидания зависимостей
- **Ambassador** — прокси для внешних сервисов

## Частые ошибки

- `CrashLoopBackOff` — контейнер падает циклически. Смотри логи: `kubectl logs <pod>`
- `ImagePullBackOff` — не удалось скачать образ. Проверь имя образа и `imagePullSecrets`
- `OOMKilled` — превышен лимит памяти. Увеличь `resources.limits.memory`
