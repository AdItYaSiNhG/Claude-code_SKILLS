---
name: backend-database
description: Database schema design, query optimisation, migration management, and modern data stack (dbt, Kafka, Airflow, Spark). Use for OLTP/OLAP decisions, index strategy, large-table migrations, and data pipeline architecture.
---

The user is working on database design, performance, or data engineering. Apply the relevant section below.

## Database Selection

| Workload | Recommended |
|---|---|
| General OLTP, relational | PostgreSQL (default choice) |
| High-write, low-latency | MySQL 8+ or ScyllaDB |
| Document / flexible schema | MongoDB or PostgreSQL JSONB |
| Time-series | TimescaleDB (Postgres extension) or InfluxDB |
| Search | Elasticsearch / OpenSearch (complement, not replace OLTP DB) |
| Analytical (OLAP) | Snowflake, BigQuery, DuckDB, or ClickHouse |
| Graph | Neo4j or AWS Neptune |
| Vector (RAG/ML) | pgvector, Pinecone, Weaviate |

## Schema Design

### Naming Conventions (PostgreSQL)
```sql
-- ✅ snake_case, plural tables, singular columns, explicit FK names
CREATE TABLE orders (
  id          UUID        DEFAULT gen_random_uuid() PRIMARY KEY,
  customer_id UUID        NOT NULL REFERENCES customers(id),
  status      TEXT        NOT NULL CHECK (status IN ('pending','confirmed','shipped','cancelled')),
  total_cents INTEGER     NOT NULL CHECK (total_cents >= 0),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- ✅ Enum as CHECK constraint or pg native enum
-- Use CHECK for simple lists; pg enum for join-needed lookups
```

### Soft Delete Pattern
```sql
-- Add to tables requiring audit trail
deleted_at TIMESTAMPTZ DEFAULT NULL

-- Partial index keeps active-record queries fast
CREATE INDEX idx_orders_active ON orders (customer_id)
WHERE deleted_at IS NULL;
```

## Indexing Strategy

```sql
-- B-tree (default): equality + range on orderable types
CREATE INDEX idx_orders_customer_created ON orders (customer_id, created_at DESC);

-- Partial index: subset of rows (most selective)
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';

-- GIN: JSONB, array, full-text search
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_orders_metadata ON orders USING GIN (metadata jsonb_path_ops);

-- BRIN: append-only time-series tables (tiny index, fast scans)
CREATE INDEX idx_events_time ON events USING BRIN (created_at);
```

### Diagnosing slow queries
```sql
-- Enable auto_explain
ALTER SYSTEM SET auto_explain.log_min_duration = '100ms';

-- Analyse a specific query
EXPLAIN (ANALYSE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 20;

-- Find missing indexes
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats WHERE tablename = 'orders';

-- Top 10 slowest queries (pg_stat_statements)
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;
```

## Migration Management

### Safe Large-Table Migrations (zero downtime)
```sql
-- ❌ NEVER: ALTER TABLE orders ADD COLUMN discount_cents INTEGER NOT NULL DEFAULT 0;
-- Locks the table for minutes on large tables

-- ✅ Step 1: Add nullable column (instant, no lock)
ALTER TABLE orders ADD COLUMN discount_cents INTEGER;

-- ✅ Step 2: Backfill in batches (background job)
UPDATE orders SET discount_cents = 0
WHERE discount_cents IS NULL AND id BETWEEN $start AND $end;

-- ✅ Step 3: Add NOT NULL constraint with default (Postgres 11+, fast)
ALTER TABLE orders ALTER COLUMN discount_cents SET DEFAULT 0;
ALTER TABLE orders ALTER COLUMN discount_cents SET NOT NULL;

-- ✅ Step 4: Remove default if not needed at DB level
ALTER TABLE orders ALTER COLUMN discount_cents DROP DEFAULT;
```

### Index creation (no downtime)
```sql
-- Always use CONCURRENTLY
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
-- Never use CONCURRENTLY in a transaction block
```

### Migration Tools
```
Flyway  → Java/enterprise, strong for multi-team governance, versioned SQL files
Liquibase → XML/YAML/SQL, rollback support, good for compliance audit trails
Prisma Migrate → TypeScript ORMs, dev-friendly, generates SQL diff
Alembic → Python/SQLAlchemy
golang-migrate → Go projects
```

## Modern Data Stack

### dbt (Data Build Tool)
```sql
-- models/marts/orders/fct_orders.sql
{{ config(materialized='incremental', unique_key='order_id') }}

SELECT
  o.id                AS order_id,
  o.customer_id,
  o.total_cents / 100.0 AS total_usd,
  DATE_TRUNC('day', o.created_at) AS order_date,
  c.segment           AS customer_segment
FROM {{ ref('stg_orders') }} o
JOIN {{ ref('dim_customers') }} c ON o.customer_id = c.id
{% if is_incremental() %}
WHERE o.created_at > (SELECT MAX(order_date) FROM {{ this }})
{% endif %}
```

Key dbt practices:
- Staging → Intermediate → Mart layering (never raw → mart directly)
- `dbt test` for not_null, unique, accepted_values, relationships
- Source freshness checks on every ingested table
- Docs: `dbt docs generate && dbt docs serve`

### Kafka (Event Streaming)
```
Topic naming: {domain}.{entity}.{event}  →  orders.order.created
Partitioning: by entity ID for ordering guarantees per entity
Retention:    7 days default, 30 days for compliance-sensitive topics
Consumer groups: one per downstream service
Exactly-once:  use transactions + idempotent producer for financial data
```

### Airflow DAG Pattern
```python
with DAG('orders_daily_sync', schedule='@daily', catchup=False) as dag:
    extract = PythonOperator(task_id='extract', python_callable=extract_orders)
    transform = PythonOperator(task_id='transform', python_callable=transform)
    load = PythonOperator(task_id='load', python_callable=load_to_warehouse)
    extract >> transform >> load
```

## Output Format

1. For schema questions: show DDL with constraints and index strategy explained
2. For slow query: show EXPLAIN output interpretation + index fix
3. For migrations: always show the safe zero-downtime approach, not the naive one
4. For data stack: show the tool config snippet + explain where it fits in the pipeline
