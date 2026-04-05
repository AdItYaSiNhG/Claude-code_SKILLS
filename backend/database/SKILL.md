---
name: backend-database
description: >
  Use this skill for all database tasks: PostgreSQL query optimisation, MySQL/MariaDB,
  schema migrations (Flyway/Liquibase/Prisma), query plan analysis, index recommendations,
  MongoDB, Redis caching patterns, Cassandra, Elasticsearch, dbt transformations,
  Apache Kafka event streaming, Airflow/Prefect DAGs, Spark/Flink pipelines, and
  data warehouse work (Snowflake/BigQuery/Redshift).
  Triggers: "database", "PostgreSQL", "MySQL", "MongoDB", "Redis", "Elasticsearch",
  "Cassandra", "schema migration", "query optimisation", "index", "EXPLAIN",
  "slow query", "dbt", "Kafka", "Airflow", "data pipeline", "warehouse", "Snowflake".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# Database & Data Engineering Skill

## Why this skill exists

Database problems are the most expensive performance issues to fix post-launch.
This skill provides **query analysis, index strategy, migration patterns, caching
architecture, and data pipeline patterns** across the full modern data stack —
from transactional OLTP to streaming pipelines and warehouses.

---

## 0. Database audit

```bash
# Detect databases in use
grep -rn "postgres\|mysql\|mongodb\|redis\|elasticsearch\|cassandra\|snowflake\|bigquery" \
  --include="*.ts" --include="*.py" --include="*.go" --include="*.env*" \
  . | grep -v "node_modules\|test" | grep -i "connect\|url\|host" | head -20

# ORM detection
cat package.json 2>/dev/null | python3 -c "
import json,sys; d=json.load(sys.stdin)
orms = ['prisma','drizzle-orm','typeorm','sequelize','knex','mongoose','@prisma/client']
all_deps = {**d.get('dependencies',{}), **d.get('devDependencies',{})}
for o in orms:
    if o in all_deps: print(f'{o}: {all_deps[o]}')
"

grep -rn "SQLAlchemy\|Tortoise\|Peewee\|Django.*models" --include="*.py" . 2>/dev/null | head -5
```

---

## 1. PostgreSQL — query optimisation

### EXPLAIN ANALYZE workflow
```sql
-- Always run on slow queries (>100ms in production)
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT u.id, u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name
ORDER BY order_count DESC
LIMIT 20;

-- Red flags to look for:
-- Seq Scan on large table (>10k rows) → needs index
-- Hash Join with high rows estimate → may need index on join column
-- Sort (cost>1000) → covering index or materialised view
-- Rows Removed by Filter: high % → index selectivity problem
```

### Index strategy
```sql
-- B-tree (default) — equality + range queries
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
CREATE INDEX CONCURRENTLY idx_orders_user_created ON orders(user_id, created_at DESC);

-- Partial index — only index the rows you query
CREATE INDEX CONCURRENTLY idx_orders_pending ON orders(created_at)
  WHERE status = 'pending';

-- Composite — column order matters: put equality first, range last
-- Query: WHERE tenant_id = $1 AND created_at > $2
CREATE INDEX idx_events_tenant_time ON events(tenant_id, created_at DESC);

-- Covering index — include extra columns to avoid table lookup
CREATE INDEX idx_users_email_covering ON users(email)
  INCLUDE (id, name, role);

-- GIN index — full-text search, JSONB, arrays
CREATE INDEX idx_products_search ON products USING gin(to_tsvector('english', name || ' ' || description));
CREATE INDEX idx_metadata ON events USING gin(metadata);  -- JSONB queries

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND schemaname = 'public'  -- unused indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (high sequential scans)
SELECT relname, seq_scan, seq_tup_read, idx_scan,
       seq_tup_read / NULLIF(seq_scan, 0) as avg_rows_per_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100 AND seq_tup_read > 10000
ORDER BY seq_tup_read DESC LIMIT 20;
```

### N+1 query detection and fix
```typescript
// PROBLEM — N+1: 1 query for users + N queries for orders
const users = await db.user.findMany();
for (const user of users) {
  user.orders = await db.order.findMany({ where: { userId: user.id } }); // N queries!
}

// FIX — single query with include
const users = await db.user.findMany({
  include: { orders: { orderBy: { createdAt: "desc" }, take: 5 } },
});

// FIX — dataloader pattern (for GraphQL)
import DataLoader from "dataloader";
const orderLoader = new DataLoader<string, Order[]>(async (userIds) => {
  const orders = await db.order.findMany({ where: { userId: { in: [...userIds] } } });
  const byUser = new Map<string, Order[]>();
  orders.forEach(o => { const arr = byUser.get(o.userId) ?? []; arr.push(o); byUser.set(o.userId, arr); });
  return userIds.map(id => byUser.get(id) ?? []);
});
```

### Schema migrations (Prisma)
```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String
  role      Role     @default(USER)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  orders    Order[]

  @@index([createdAt(sort: Desc)])
  @@index([role])
  @@map("users")
}

model Order {
  id        String      @id @default(cuid())
  userId    String      @map("user_id")
  status    OrderStatus @default(PENDING)
  total     Decimal     @db.Decimal(10, 2)
  createdAt DateTime    @default(now()) @map("created_at")
  user      User        @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId, createdAt(sort: Desc)])
  @@index([status, createdAt])
  @@map("orders")
}

enum Role        { USER ADMIN VIEWER }
enum OrderStatus { PENDING PROCESSING SHIPPED DELIVERED CANCELLED }
```

```bash
# Migration workflow
npx prisma migrate dev --name add_user_role_index  # development
npx prisma migrate deploy                           # production (CI/CD)
npx prisma db pull                                  # introspect existing DB
npx prisma studio                                   # GUI explorer
```

---

## 2. Redis caching patterns

```typescript
// lib/cache.ts — typed cache wrapper
import { Redis } from "ioredis";

const redis = new Redis(env.REDIS_URL);

export const cache = {
  // Cache-aside pattern
  async get<T>(key: string): Promise<T | null> {
    const raw = await redis.get(key);
    return raw ? JSON.parse(raw) as T : null;
  },

  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    await redis.setex(key, ttlSeconds, JSON.stringify(value));
  },

  async getOrSet<T>(key: string, fetcher: () => Promise<T>, ttl: number): Promise<T> {
    const cached = await this.get<T>(key);
    if (cached !== null) return cached;
    const fresh = await fetcher();
    await this.set(key, fresh, ttl);
    return fresh;
  },

  async invalidate(pattern: string): Promise<void> {
    const keys = await redis.keys(pattern);
    if (keys.length) await redis.del(...keys);
  },

  // Distributed lock (prevents cache stampede)
  async withLock<T>(key: string, ttl: number, fn: () => Promise<T>): Promise<T> {
    const lockKey = `lock:${key}`;
    const acquired = await redis.set(lockKey, "1", "EX", ttl, "NX");
    if (!acquired) throw new Error("Lock not acquired");
    try { return await fn(); }
    finally { await redis.del(lockKey); }
  },
};

// Cache key conventions
export const cacheKeys = {
  user:     (id: string) => `user:${id}`,
  userList: (filters: string) => `users:list:${filters}`,
  session:  (sessionId: string) => `session:${sessionId}`,
  rateLimit:(ip: string, endpoint: string) => `rate:${ip}:${endpoint}`,
};

// Usage
const user = await cache.getOrSet(
  cacheKeys.user(userId),
  () => db.user.findUnique({ where: { id: userId } }),
  300  // 5 min TTL
);
```

```bash
# Redis health checks
redis-cli ping 2>/dev/null
redis-cli info memory 2>/dev/null | grep "used_memory_human\|maxmemory_human"
redis-cli info stats 2>/dev/null | grep "keyspace_hits\|keyspace_misses"
# Hit rate = keyspace_hits / (keyspace_hits + keyspace_misses)
# Target: > 80%
```

---

## 3. MongoDB

```typescript
// src/domains/products/productRepository.ts
import { MongoClient, ObjectId, type Collection } from "mongodb";

export class ProductRepository {
  private collection: Collection<ProductDocument>;

  constructor(db: Db) {
    this.collection = db.collection<ProductDocument>("products");
  }

  async findById(id: string): Promise<Product | null> {
    const doc = await this.collection.findOne({ _id: new ObjectId(id) });
    return doc ? this.toDomain(doc) : null;
  }

  async search(query: ProductSearchQuery): Promise<{ items: Product[]; total: number }> {
    const filter: Filter<ProductDocument> = {};
    if (query.category)  filter.category = query.category;
    if (query.minPrice)  filter.price = { $gte: query.minPrice };
    if (query.maxPrice)  filter.price = { ...filter.price as object, $lte: query.maxPrice };
    if (query.q) filter.$text = { $search: query.q };

    const [items, total] = await Promise.all([
      this.collection
        .find(filter)
        .sort({ score: { $meta: "textScore" }, createdAt: -1 })
        .skip((query.page - 1) * query.limit)
        .limit(query.limit)
        .toArray(),
      this.collection.countDocuments(filter),
    ]);

    return { items: items.map(this.toDomain), total };
  }
}
```

```javascript
// MongoDB indexes
db.products.createIndex({ category: 1, price: 1 });           // compound
db.products.createIndex({ name: "text", description: "text" }); // full-text
db.products.createIndex({ tenantId: 1, createdAt: -1 });      // multi-tenant sort
db.products.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 }); // TTL
```

---

## 4. Elasticsearch

```typescript
// src/search/productSearch.ts
import { Client } from "@elastic/elasticsearch";

const es = new Client({ node: env.ELASTICSEARCH_URL });

export async function indexProduct(product: Product): Promise<void> {
  await es.index({
    index: "products",
    id:    product.id,
    document: {
      name:        product.name,
      description: product.description,
      category:    product.category,
      price:       product.price,
      tags:        product.tags,
      inStock:     product.stock > 0,
      createdAt:   product.createdAt,
    },
  });
}

export async function searchProducts(query: SearchQuery) {
  const response = await es.search({
    index: "products",
    body: {
      query: {
        bool: {
          must: [
            query.q ? { multi_match: { query: query.q, fields: ["name^3", "description", "tags"], fuzziness: "AUTO" } } : { match_all: {} },
          ],
          filter: [
            query.category ? { term: { category: query.category } } : null,
            query.inStock   ? { term: { inStock: true } } : null,
            query.minPrice || query.maxPrice ? { range: { price: { gte: query.minPrice, lte: query.maxPrice } } } : null,
          ].filter(Boolean),
        },
      },
      sort: [{ _score: "desc" }, { createdAt: "desc" }],
      from: (query.page - 1) * query.limit,
      size: query.limit,
      highlight: { fields: { name: {}, description: { fragment_size: 150 } } },
    },
  });

  return {
    items: response.hits.hits.map(h => ({ ...h._source, id: h._id, highlight: h.highlight })),
    total: (response.hits.total as any).value,
  };
}
```

---

## 5. Apache Kafka

```typescript
// src/messaging/kafka.ts
import { Kafka, logLevel } from "kafkajs";

const kafka = new Kafka({
  clientId: "myapp",
  brokers:  env.KAFKA_BROKERS.split(","),
  ssl:      true,
  logLevel: logLevel.WARN,
});

// Producer
export const producer = kafka.producer({ idempotent: true }); // exactly-once semantics

export async function publishEvent<T>(topic: string, key: string, payload: T): Promise<void> {
  await producer.send({
    topic,
    messages: [{
      key,
      value: JSON.stringify(payload),
      headers: {
        "event-type":    topic,
        "produced-at":  new Date().toISOString(),
        "schema-version": "1",
      },
    }],
  });
}

// Consumer
export function createConsumer(groupId: string) {
  const consumer = kafka.consumer({ groupId, sessionTimeout: 30_000 });

  return {
    async subscribe(topics: string[], handler: (topic: string, key: string, value: unknown) => Promise<void>) {
      await consumer.connect();
      await consumer.subscribe({ topics, fromBeginning: false });

      await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
          const key   = message.key?.toString() ?? "";
          const value = JSON.parse(message.value?.toString() ?? "{}");
          try {
            await handler(topic, key, value);
          } catch (err) {
            // Dead letter queue — publish to DLQ for manual inspection
            await publishEvent(`${topic}.dlq`, key, { original: value, error: (err as Error).message, partition });
            logger.error({ topic, key, err }, "Message processing failed → DLQ");
          }
        },
      });
    },
  };
}
```

---

## 6. dbt transformations

```sql
-- models/marts/finance/fct_orders.sql
{{
  config(
    materialized = 'incremental',
    unique_key    = 'order_id',
    on_schema_change = 'sync_all_columns',
    tags = ['finance', 'daily']
  )
}}

WITH source_orders AS (
    SELECT * FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
      WHERE updated_at > (SELECT MAX(updated_at) FROM {{ this }})
    {% endif %}
),

enriched AS (
    SELECT
        o.order_id,
        o.user_id,
        u.segment         AS customer_segment,
        o.total_amount,
        o.status,
        o.created_at,
        o.updated_at,
        DATE_TRUNC('month', o.created_at) AS order_month,
        CASE
            WHEN o.total_amount >= 500 THEN 'large'
            WHEN o.total_amount >= 100 THEN 'medium'
            ELSE 'small'
        END AS order_size
    FROM source_orders o
    LEFT JOIN {{ ref('dim_users') }} u USING (user_id)
    WHERE o.status != 'cancelled'
)

SELECT * FROM enriched
```

```yaml
# models/marts/finance/fct_orders.yml
version: 2
models:
  - name: fct_orders
    description: "Fact table for all non-cancelled orders"
    columns:
      - name: order_id
        description: "Primary key"
        tests: [unique, not_null]
      - name: total_amount
        tests: [not_null, { dbt_utils.accepted_range: { min_value: 0 } }]
      - name: status
        tests: [{ accepted_values: { values: ['pending','processing','shipped','delivered'] } }]
```

```bash
# dbt workflow
dbt run --select fct_orders 2>/dev/null
dbt test --select fct_orders 2>/dev/null
dbt run --select tag:finance --target prod 2>/dev/null
dbt docs generate && dbt docs serve 2>/dev/null
```

---

## 7. Airflow DAG

```python
# dags/daily_user_metrics.py
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.slack.operators.slack_webhook import SlackWebhookOperator

default_args = {
    "owner":            "data-team",
    "retries":          2,
    "retry_delay":      timedelta(minutes=5),
    "email_on_failure": True,
    "email":            ["data-alerts@myapp.com"],
}

with DAG(
    dag_id="daily_user_metrics",
    default_args=default_args,
    schedule_interval="0 2 * * *",    # 2am UTC daily
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=["metrics", "users"],
    doc_md="Computes daily user engagement metrics and loads to warehouse.",
) as dag:

    extract_task = PostgresOperator(
        task_id="extract_events",
        postgres_conn_id="prod_db",
        sql="sql/extract_daily_events.sql",
        parameters={"date": "{{ ds }}"},
    )

    transform_task = PythonOperator(
        task_id="transform_metrics",
        python_callable=compute_user_metrics,
        op_kwargs={"date": "{{ ds }}"},
    )

    load_task = PythonOperator(
        task_id="load_to_warehouse",
        python_callable=load_to_bigquery,
        op_kwargs={"date": "{{ ds }}", "table": "user_daily_metrics"},
    )

    notify_task = SlackWebhookOperator(
        task_id="notify_success",
        slack_webhook_conn_id="slack_data_alerts",
        message="✅ Daily user metrics loaded for {{ ds }}",
        trigger_rule="all_success",
    )

    extract_task >> transform_task >> load_task >> notify_task
```

---

## 8. Slow query detection

```bash
# PostgreSQL — enable slow query logging
# In postgresql.conf:
# log_min_duration_statement = 1000  (ms)
# log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Find slow queries from pg_stat_statements
psql $DATABASE_URL -c "
SELECT
    round(mean_exec_time)   AS avg_ms,
    round(max_exec_time)    AS max_ms,
    calls,
    round(total_exec_time)  AS total_ms,
    query
FROM pg_stat_statements
WHERE mean_exec_time > 100
ORDER BY mean_exec_time DESC
LIMIT 20;
" 2>/dev/null

# Find missing indexes (high cost sequential scans)
psql $DATABASE_URL -c "
SELECT
    relname           AS table,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_seq_rows
FROM pg_stat_user_tables
WHERE seq_scan > 50
ORDER BY seq_tup_read DESC
LIMIT 15;
" 2>/dev/null

# Table bloat estimate
psql $DATABASE_URL -c "
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total_size,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_size,
    n_dead_tup, n_live_tup,
    round(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
" 2>/dev/null
```

---

## 9. Data pipeline — connection management

```typescript
// src/db/pool.ts — connection pool best practices
import { Pool } from "pg";

export const pool = new Pool({
  connectionString: env.DATABASE_URL,
  max:              env.NODE_ENV === "production" ? 20 : 5,
  min:              2,
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000,
  statement_timeout: 30_000,    // kill runaway queries after 30s
  query_timeout:    30_000,
});

// Graceful shutdown
process.on("SIGTERM", async () => {
  await pool.end();
});

// Health check
pool.on("error", (err) => {
  logger.error({ err }, "Unexpected error on idle pool client");
});
```

---

## 10. Migration safety checklist

```
Before every production migration:
  [ ] Migration is backwards-compatible (app v_N-1 works with schema v_N)
  [ ] Large table changes use CONCURRENTLY or batched approach
  [ ] Estimated migration duration tested on prod-sized dataset
  [ ] Rollback SQL written and tested

Safe patterns:
  ADD COLUMN (nullable or with DEFAULT) → safe, instant
  ADD INDEX CONCURRENTLY               → safe, no table lock
  DROP COLUMN                          → safe after deploy removes all references
  RENAME COLUMN                        → UNSAFE — use add + copy + drop instead
  ALTER COLUMN type                    → UNSAFE on large tables — use shadow column
  ADD NOT NULL constraint              → UNSAFE without default — add DEFAULT first

Batched update (avoid long-running lock):
  UPDATE users SET new_col = old_col WHERE id IN (
    SELECT id FROM users WHERE new_col IS NULL LIMIT 1000
  );
  -- Run in a loop until 0 rows updated
```
