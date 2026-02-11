# Alerting

Уведомления о проблемах и аномалиях в работе инстансов.

## Два уровня алертов

### Platform alerts (внутренние)

Для команды DBaaS. Проблемы инфраструктуры, Control Plane, Data Plane.

```yaml
- alert: InstanceDown
  expr: up{job="postgresql"} == 0
  for: 1m
  labels:
    severity: critical
    team: dbaas-platform

- alert: ControlPlaneHighLatency
  expr: histogram_quantile(0.99, rate(cp_request_duration_seconds_bucket[5m])) > 5
  for: 5m
  labels:
    severity: warning
    team: dbaas-platform
```

### User-facing alerts (для пользователей)

Алерты о состоянии конкретных инстансов, видимые пользователю.

```json
POST /v1/instances/{id}/alerts
{
  "name": "High CPU Usage",
  "condition": {
    "metric": "cpu_usage_percent",
    "operator": ">",
    "threshold": 80,
    "duration": "10m"
  },
  "channels": ["email", "slack"],
  "recipients": {
    "email": ["ops@company.com"],
    "slack_webhook": "https://hooks.slack.com/..."
  }
}
```

## Критические алерты (must-have)

| Алерт | Условие | Severity | Действие |
|-------|---------|----------|----------|
| Instance down | health check fails 3x | Critical | Auto-failover |
| Disk almost full | > 90% usage | Critical | Auto-expand или alert |
| Replication broken | replication state ≠ streaming | Critical | Investigate |
| High replication lag | lag > 60s | Warning | Investigate |
| High CPU | > 90% за 15 мин | Warning | Scale up |
| High memory | > 90% | Warning | Scale up / tune |
| Too many connections | > 80% max_connections | Warning | Connection pooling |
| High error rate | errors > 10/min | Warning | Check logs |
| Backup failed | backup job failed | Critical | Retry, investigate |
| Certificate expiring | < 14 days to expiry | Warning | Rotate |

## Alerting Pipeline

```
Metrics (Prometheus)
  │
  ▼
Alert Rules (Alertmanager / Grafana)
  │
  ├── Grouping (по instance, severity)
  ├── Deduplication
  ├── Silencing (maintenance window)
  ├── Inhibition (critical подавляет warning)
  │
  ▼
Notification channels
  ├── Email
  ├── Slack / Telegram
  ├── PagerDuty / OpsGenie
  ├── Webhook
  └── In-app notification
```

## Alertmanager config

```yaml
route:
  group_by: ['instance_id', 'alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      repeat_interval: 15m
    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: '<key>'
  - name: 'slack'
    slack_configs:
      - channel: '#dbaas-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: 'Instance: {{ .GroupLabels.instance_id }}'
```

## Maintenance Window Silencing

Подавление алертов во время planned maintenance.

```yaml
# Silence для конкретного инстанса
silence:
  matchers:
    - name: instance_id
      value: inst-abc123
  startsAt: "2026-02-15T03:00:00Z"
  endsAt: "2026-02-15T04:00:00Z"
  comment: "Planned maintenance: version upgrade"
  createdBy: "platform-automation"
```

## Шаблон уведомления

```
🔴 CRITICAL: Instance Down

Instance: inst-abc123 (my-postgres)
Project: proj-456
Region: eu-west-1

Status: Primary not responding
Duration: 2 minutes
Action: Automatic failover initiated

Dashboard: https://console.example.com/instances/abc123
Runbook: https://wiki.internal/runbooks/instance-down
```

## Best Practices

1. **Actionable alerts** — каждый алерт должен иметь runbook / рекомендуемое действие
2. **Avoid alert fatigue** — не алертить на то, что не требует действия
3. **Tiered severity** — critical (page), warning (investigate), info (log)
4. **Deduplication** — один алерт на проблему, не десятки
5. **Auto-resolve** — алерт автоматически закрывается при восстановлении
6. **Test alerts** — периодическая проверка что алерты доходят
