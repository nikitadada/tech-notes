# Troubleshooting Kubernetes

## Алгоритм диагностики

```
Pod не работает?
  ├── kubectl get pods → смотри STATUS
  │   ├── Pending       → проблема со scheduling
  │   ├── CrashLoopBackOff → контейнер падает
  │   ├── ImagePullBackOff → проблема с образом
  │   ├── OOMKilled     → нехватка памяти
  │   └── Running, но не работает → проблема внутри приложения
  ├── kubectl describe pod <name> → Events секция
  └── kubectl logs <name> → логи приложения
```

## Pod в Pending

**Причины:**
- Нехватка ресурсов на узлах (CPU/Memory)
- Нет узлов с нужными taints/tolerations
- PVC не может быть привязан к PV

**Диагностика:**

```bash
kubectl describe pod <name>
# Смотри секцию Events: "FailedScheduling"

kubectl describe nodes | grep -A5 "Allocated resources"
kubectl get pvc  # если есть volumeMounts
```

## CrashLoopBackOff

Pod стартует и сразу падает. Kubernetes перезапускает с exponential backoff.

```bash
# Логи текущего контейнера
kubectl logs <pod-name>

# Логи предыдущего запуска
kubectl logs <pod-name> --previous

# Запустить pod с командой sleep для отладки
kubectl run debug --image=<same-image> --command -- sleep 3600
kubectl exec -it debug -- /bin/sh
```

**Частые причины:**
- Ошибка в конфигурации (переменные окружения, файлы)
- Приложение не может подключиться к зависимости (БД, кэш)
- Ошибка в entrypoint/command
- Неправильный health check

## ImagePullBackOff

```bash
kubectl describe pod <name>
# Ищи: "Failed to pull image"
```

**Причины:**
- Неверное имя или тег образа
- Нет доступа к private registry (`imagePullSecrets`)
- Rate limit Docker Hub

## OOMKilled

Контейнер превысил лимит памяти.

```bash
kubectl describe pod <name> | grep -A3 "Last State"
# Reason: OOMKilled

kubectl top pod <name>  # текущее потребление
```

**Решение:** увеличить `resources.limits.memory` или найти утечку памяти.

## Service не работает

```bash
# 1. Проверить endpoints
kubectl get endpoints <service-name>
# Пустой? → selector не совпадает с labels Pod'ов

# 2. Проверить selector
kubectl get svc <service-name> -o yaml | grep -A5 selector
kubectl get pods -l <selector-key>=<selector-value>

# 3. Проверить доступность из другого Pod'а
kubectl run curl --image=curlimages/curl --rm -it -- curl http://<service-name>:<port>

# 4. DNS
kubectl run dns-test --image=busybox --rm -it -- nslookup <service-name>
```

## Node проблемы

```bash
kubectl get nodes
kubectl describe node <name>

# Conditions
# Ready=False → kubelet не отвечает
# MemoryPressure=True → мало памяти
# DiskPressure=True → мало диска
# PIDPressure=True → много процессов

# Проверить на самом узле
journalctl -u kubelet -f
```

## Сетевые проблемы

```bash
# Проверить connectivity между Pod'ами
kubectl exec -it <pod-a> -- ping <pod-b-ip>
kubectl exec -it <pod-a> -- wget -qO- http://<service>:<port>

# Проверить NetworkPolicy
kubectl get networkpolicies -n <namespace>

# DNS debug
kubectl exec -it <pod> -- nslookup kubernetes.default
```

## Полезные инструменты

- `kubectl debug` — создание debug-контейнера в running Pod
- `kubectl events` — просмотр событий кластера
- `stern` — агрегация логов нескольких Pod'ов
- `k9s` — TUI для управления кластером
- `kubectx` / `kubens` — быстрое переключение контекстов/namespaces
