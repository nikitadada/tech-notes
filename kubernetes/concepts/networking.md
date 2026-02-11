# Networking в Kubernetes

## Сетевая модель

Kubernetes требует, чтобы:
1. Каждый Pod имел уникальный IP
2. Pod'ы могли общаться друг с другом без NAT
3. Агенты на узле (kubelet, kube-proxy) могли общаться со всеми Pod'ами

Реализация — через CNI-плагины (Container Network Interface).

## CNI-плагины

| Плагин | Особенности |
|--------|------------|
| Calico | NetworkPolicy, BGP, высокая производительность |
| Cilium | eBPF, L7 policies, observability |
| Flannel | Простой overlay (VXLAN), без NetworkPolicy |
| Weave | Mesh-сеть, шифрование |

## Network Policies

Файрвол на уровне Pod'ов. По умолчанию весь трафик разрешён — NetworkPolicy ограничивает его.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: production
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
```

Это правило: api-server принимает входящий трафик только от frontend Pod'ов в production namespace на порт 8080, и может ходить только в postgres на порт 5432.

## Ingress

Маршрутизация внешнего HTTP/HTTPS-трафика к сервисам внутри кластера.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

## DNS в кластере

CoreDNS — DNS-сервер кластера. Разрешает имена:

```
<service>.<namespace>.svc.cluster.local    → ClusterIP Service
<pod-ip-dashed>.<namespace>.pod.cluster.local  → Pod напрямую
```

## Service Mesh

Sidecar-прокси для управления трафиком между сервисами:

| Решение | Описание |
|---------|----------|
| Istio | Полнофункциональный, Envoy-based |
| Linkerd | Легковесный, Rust-based прокси |
| Cilium | eBPF-based, без sidecar |

Возможности: mTLS, трейсинг, circuit breaking, traffic splitting, retries.
