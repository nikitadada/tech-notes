# Major DBaaS Vendors

## Cloud-провайдеры

### AWS

| Сервис | СУБД | Особенности |
|--------|------|-------------|
| RDS | PostgreSQL, MySQL, MariaDB, Oracle, SQL Server | Самый зрелый managed DB |
| Aurora | MySQL, PostgreSQL совместимая | Распределённый storage, до 5x быстрее |
| DynamoDB | Key-Value / Document | Serverless, single-digit ms latency |
| ElastiCache | Redis, Memcached | In-memory cache |
| MemoryDB | Redis-совместимая | Durable in-memory DB |
| Redshift | Columnar (аналитика) | Petabyte-scale OLAP |

### GCP

| Сервис | СУБД | Особенности |
|--------|------|-------------|
| Cloud SQL | PostgreSQL, MySQL, SQL Server | Простой managed DB |
| AlloyDB | PostgreSQL-совместимая | Disaggregated storage, аналитика |
| Cloud Spanner | NewSQL (proprietary) | Global distribution, strong consistency |
| Bigtable | Wide-column | High-throughput, HBase-совместимая |
| Firestore | Document | Serverless NoSQL |

### Azure

| Сервис | СУБД | Особенности |
|--------|------|-------------|
| Azure Database | PostgreSQL, MySQL, MariaDB | Flexible Server |
| Cosmos DB | Multi-model | Global distribution, 5 consistency levels |
| SQL Database | SQL Server | Hyperscale tier |

## Независимые провайдеры

| Провайдер | СУБД | Особенности |
|-----------|------|-------------|
| Neon | PostgreSQL | Serverless, branching, scale-to-zero |
| Supabase | PostgreSQL | Open-source Firebase alternative |
| PlanetScale | MySQL (Vitess) | Branching, non-blocking schema changes |
| MongoDB Atlas | MongoDB | Multi-cloud, serverless tier |
| CockroachDB Cloud | CockroachDB | Distributed SQL, multi-region |
| Aiven | PG, MySQL, Redis, Kafka, etc. | Multi-cloud, open-source stack |
| Timescale Cloud | TimescaleDB | Time-series optimized PostgreSQL |
| ClickHouse Cloud | ClickHouse | Serverless analytics |

## Kubernetes Operators (self-hosted DBaaS)

| Оператор | СУБД | Описание |
|----------|------|----------|
| CloudNativePG | PostgreSQL | CNCF, production-ready |
| Zalando Postgres Operator | PostgreSQL | Patroni-based, зрелый |
| Percona Operator | PG, MySQL, MongoDB | Multi-DB, Percona backup |
| Vitess Operator | MySQL (Vitess) | Sharding, horizontal scaling |
| Strimzi | Kafka | Kafka on Kubernetes |
| Redis Operator (Spotahome) | Redis | Redis Cluster / Sentinel |

## Критерии выбора

| Критерий | Вопрос |
|----------|--------|
| Lock-in | Насколько легко мигрировать? Proprietary vs стандартная СУБД? |
| Multi-cloud | Работает только в одном облаке или нескольких? |
| Pricing | Pay-per-use vs reserved? Стоимость storage/compute? |
| Compliance | SOC 2, HIPAA, GDPR, PCI DSS? |
| HA / DR | Автоматический failover? Cross-region replication? |
| Serverless | Scale-to-zero? Подходит для spiky workloads? |
| Ecosystem | Мониторинг, бэкапы, connection pooling — встроено или нужно добавлять? |

## Тренды

- **Serverless** — Neon, Aurora Serverless v2, ClickHouse Cloud — оплата за фактическое использование
- **Branching** — git-like workflow для схемы и данных (Neon, PlanetScale)
- **Disaggregated storage** — compute и storage масштабируются независимо (Aurora, AlloyDB)
- **AI-интеграция** — pgvector, встроенные embedding'и, vector search
- **Multi-cloud** — CockroachDB, MongoDB Atlas — один кластер в нескольких облаках
