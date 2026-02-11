# Metrics

Количественные измерения состояния и производительности инстансов БД.

## Ключевые метрики

### System-level

| Метрика | Описание | Alert threshold |
|---------|----------|-----------------|
| `cpu_usage_percent` | Загрузка CPU | > 80% за 10 мин |
| `memory_usage_percent` | Использование RAM | > 85% |
| `disk_usage_percent` | Заполненность диска | > 80% |
| `disk_iops` | I/O operations per second | Зависит от плана |
| `disk_throughput_mbps` | Пропускная способность диска | Зависит от плана |
| `network_rx_bytes` | Входящий трафик | Anomaly detection |
| `network_tx_bytes` | Исходящий трафик | Anomaly detection |

### Database-level (PostgreSQL)

| Метрика | Описание | Источник |
|---------|----------|----------|
| `pg_connections_active` | Активные соединения | `pg_stat_activity` |
| `pg_connections_idle` | Простаивающие соединения | `pg_stat_activity` |
| `pg_connections_max` | Максимум соединений | `max_connections` |
| `pg_transactions_per_sec` | TPS | `pg_stat_database` |
| `pg_queries_per_sec` | QPS | `pg_stat_statements` |
| `pg_query_duration_p99` | p99 latency запросов | `pg_stat_statements` |
| `pg_cache_hit_ratio` | % попаданий в buffer cache | `pg_stat_database` |
| `pg_replication_lag_sec` | Отставание реплики | `pg_stat_replication` |
| `pg_deadlocks_total` | Количество deadlocks | `pg_stat_database` |
| `pg_temp_files_bytes` | Temp files (мало work_mem) | `pg_stat_database` |
| `pg_table_bloat_percent` | Раздутие таблиц (нужен VACUUM) | Расчётная |

### Slow queries

```sql
-- Top 10 запросов по общему времени
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

## Архитектура сбора метрик

```
Instance:
  DB → postgres_exporter → /metrics endpoint
  OS → node_exporter → /metrics endpoint
  
Scraping:
  Prometheus/VictoriaMetrics → scrape /metrics every 15s

Storage:
  Prometheus → Remote Write → Long-term storage (Thanos/Cortex/VM)

Visualization:
  Grafana → query Prometheus → dashboards
```

### Agent-based collection

```go
// Agent собирает метрики и отправляет в CP
func (a *Agent) collectMetrics(ctx context.Context) (*Metrics, error) {
    m := &Metrics{
        Timestamp: time.Now(),
    }

    // System metrics
    m.CPU, _ = getCPUUsage()
    m.Memory, _ = getMemoryUsage()
    m.Disk, _ = getDiskUsage()

    // DB metrics
    m.Connections, _ = a.db.Query("SELECT count(*) FROM pg_stat_activity")
    m.ReplicationLag, _ = a.db.Query("SELECT extract(epoch from replay_lag) FROM pg_stat_replication")
    m.TPS, _ = a.db.Query("SELECT xact_commit + xact_rollback FROM pg_stat_database WHERE datname = current_database()")

    return m, nil
}
```

## Dashboards

### Instance Overview

```
┌──────────────────────────────────────────────┐
│ Instance: inst-abc123 | PostgreSQL 16 | Running │
├──────────────┬───────────────┬───────────────┤
│ CPU: 45%     │ Memory: 62%   │ Disk: 38%     │
│ ████▌        │ ██████▏       │ ███▊          │
├──────────────┼───────────────┼───────────────┤
│ Connections  │ TPS           │ Cache Hit     │
│ 42 / 200     │ 1,250         │ 99.2%         │
├──────────────┴───────────────┴───────────────┤
│ Replication Lag: 0.3s | Deadlocks: 0        │
│ Slow Queries (>1s): 3 in last hour           │
└──────────────────────────────────────────────┘
```

### Рекомендуемые панели Grafana

1. **System**: CPU, Memory, Disk usage over time
2. **Connections**: active, idle, max, waiting
3. **Performance**: TPS, QPS, latency percentiles
4. **Cache**: buffer hit ratio, shared buffers usage
5. **Replication**: lag (seconds, bytes), state
6. **WAL**: WAL generation rate, archive status
7. **Tables**: seq scans vs index scans, bloat, row count
8. **Slow queries**: top N by time, calls, rows

## Экспорт метрик пользователю

```
GET /v1/instances/{id}/metrics?period=1h&step=60s

Response:
{
  "metrics": [
    {
      "name": "cpu_usage_percent",
      "values": [[1707660000, 45.2], [1707660060, 47.8], ...]
    },
    ...
  ]
}
```

Или интеграция с пользовательским Prometheus (federation / remote write).
