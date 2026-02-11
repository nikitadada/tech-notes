# kubectl Cheatsheet

## Контекст и конфигурация

```bash
# Текущий контекст
kubectl config current-context

# Список контекстов
kubectl config get-contexts

# Переключение контекста
kubectl config use-context <context-name>

# Установить namespace по умолчанию
kubectl config set-context --current --namespace=production
```

## Получение информации

```bash
# Pod'ы
kubectl get pods -o wide
kubectl get pods -l app=api-server
kubectl get pods --all-namespaces
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Подробная информация
kubectl describe pod <pod-name>
kubectl describe node <node-name>

# Все ресурсы в namespace
kubectl get all -n <namespace>

# YAML манифест запущенного ресурса
kubectl get deployment <name> -o yaml

# JSON path
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
```

## Логи и отладка

```bash
# Логи Pod'а
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>    # конкретный контейнер
kubectl logs <pod-name> --previous              # предыдущий инстанс (после рестарта)
kubectl logs -f <pod-name>                      # follow
kubectl logs -l app=api-server --all-containers # все Pod'ы по label

# Exec в контейнер
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container> -- /bin/sh

# Port forward
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward svc/<service-name> 8080:80

# Events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning

# Top (требует metrics-server)
kubectl top pods
kubectl top nodes
```

## Создание и обновление

```bash
# Применить манифест
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/         # все файлы в директории
kubectl apply -f https://raw.githubusercontent.com/...  # из URL

# Создать ресурсы быстро
kubectl create deployment nginx --image=nginx:1.27 --replicas=3
kubectl create service clusterip nginx --tcp=80:80
kubectl create configmap app-config --from-file=config.yaml
kubectl create secret generic db-creds --from-literal=password=secret

# Обновить образ
kubectl set image deployment/api api=api:v2.0

# Масштабирование
kubectl scale deployment/api --replicas=5

# Autoscaling
kubectl autoscale deployment/api --min=2 --max=10 --cpu-percent=70
```

## Удаление

```bash
kubectl delete pod <pod-name>
kubectl delete -f deployment.yaml
kubectl delete deployment <name> --cascade=foreground

# Удалить всё в namespace
kubectl delete all --all -n <namespace>

# Удалить Pod с grace period 0 (force)
kubectl delete pod <name> --grace-period=0 --force
```

## Полезные флаги

```bash
--dry-run=client -o yaml    # генерация манифеста без создания
-w / --watch                # наблюдение за изменениями
--field-selector             # фильтрация по полям
-o custom-columns           # кастомные колонки вывода
```

## Быстрая генерация манифестов

```bash
# Pod
kubectl run nginx --image=nginx:1.27 --dry-run=client -o yaml > pod.yaml

# Deployment
kubectl create deployment api --image=api:v1 --dry-run=client -o yaml > deploy.yaml

# Service
kubectl expose deployment api --port=80 --target-port=8080 --dry-run=client -o yaml > svc.yaml
```
