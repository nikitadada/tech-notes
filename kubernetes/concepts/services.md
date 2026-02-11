# Services

Абстракция, обеспечивающая стабильный endpoint для набора Pod'ов. Pod'ы эфемерны — их IP меняется, Service решает эту проблему.

## Типы сервисов

### ClusterIP (default)

Доступен только внутри кластера. Получает виртуальный IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  selector:
    app: api-server
  ports:
    - port: 80          # порт Service
      targetPort: 8080   # порт контейнера
```

### NodePort

Открывает порт на каждом узле кластера (30000–32767).

```yaml
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080    # опционально, иначе назначится автоматически
```

### LoadBalancer

Создаёт внешний load balancer (в облаке). Расширяет NodePort.

```yaml
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 8443
```

### ExternalName

DNS-алиас на внешний сервис. Не проксирует трафик.

```yaml
spec:
  type: ExternalName
  externalName: db.external-provider.com
```

## Headless Service

`clusterIP: None` — не создаёт виртуальный IP. DNS возвращает IP всех Pod'ов напрямую. Используется для StatefulSet и service discovery.

```yaml
spec:
  clusterIP: None
  selector:
    app: postgres
```

## Как Service находит Pod'ы

Через `selector` — Service направляет трафик на Pod'ы с соответствующими labels.

```
Service (selector: app=api) → Endpoints → Pod'ы с label app=api
```

kube-proxy на каждом узле обновляет iptables/IPVS-правила для маршрутизации.

## Session Affinity

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
```

Направляет запросы одного клиента на один и тот же Pod.

## DNS

Kubernetes DNS автоматически создаёт записи:

```
<service-name>.<namespace>.svc.cluster.local
```

Пример: `api-service.default.svc.cluster.local`

Внутри одного namespace можно обращаться просто по имени: `api-service`.
