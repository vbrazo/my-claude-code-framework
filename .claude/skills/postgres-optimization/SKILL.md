---
name: postgres-optimization
description: Making PostgreSQL fast — index choices, reading query plans, partitioning, JSONB queries, and connection pooling
---

# Tuning PostgreSQL

## Choosing indexes

```sql
-- B-tree index for equality and range queries (default)
CREATE INDEX idx_orders_customer_id ON orders (customer_id);

-- Composite index (column order matters: equality columns first, range last)
CREATE INDEX idx_orders_status_created ON orders (status, created_at DESC);

-- Partial index (smaller, faster for filtered queries)
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';

-- Covering index (avoids table lookup entirely)
CREATE INDEX idx_users_email_name ON users (email) INCLUDE (name, avatar_url);

-- GIN index for JSONB containment queries
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- GiST index for full-text search
CREATE INDEX idx_articles_search ON articles USING GiST (
  to_tsvector('english', title || ' ' || body)
);

-- Concurrent index creation (no table lock)
CREATE INDEX CONCURRENTLY idx_large_table_col ON large_table (col);
```

## Reading a query plan

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'shipped'
  AND o.created_at > NOW() - INTERVAL '30 days'
ORDER BY o.created_at DESC
LIMIT 20;
```

What to watch for:
- A `Seq Scan` over a large table usually means an index is missing
- A `Nested Loop` with big row estimates points at a missing join index
- A `Sort` that isn't an `Index Scan` is sorting in memory or spilling to disk
- `Buffers: shared hit` vs `shared read` tells you how much came from cache

## Partitioning

```sql
CREATE TABLE events (
    id          BIGINT GENERATED ALWAYS AS IDENTITY,
    event_type  TEXT NOT NULL,
    payload     JSONB NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Index on each partition (inherited automatically in PG 11+)
CREATE INDEX ON events (created_at, event_type);
```

Partition once a table crosses ~10M rows and your queries reliably filter on the partition key.

## Working with JSONB

```sql
-- Query nested JSONB fields
SELECT * FROM products
WHERE metadata @> '{"category": "electronics"}'
  AND (metadata ->> 'price')::numeric < 500;

-- Update nested JSONB
UPDATE products
SET metadata = jsonb_set(metadata, '{stock}', to_jsonb(stock - 1))
WHERE id = 'abc';

-- Aggregate JSONB arrays
SELECT id, jsonb_array_elements_text(metadata -> 'tags') AS tag
FROM products
WHERE metadata ? 'tags';
```

## Pooling connections

```ini
# pgbouncer.ini
[databases]
app = host=localhost port=5432 dbname=app

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
min_pool_size = 5
reserve_pool_size = 5
server_idle_timeout = 300
```

Web apps want transaction-level pooling; switch to session-level only when you depend on prepared statements or temp tables.

## Finding what's slow

```sql
-- Check for slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Find unused indexes
SELECT indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

## What to avoid

- Indexing every column instead of following your real query patterns
- `SELECT *` when a handful of columns would do
- Shipping queries without confirming index usage via `EXPLAIN ANALYZE`
- Cramming large blobs into JSONB where a typed side table fits better
- Skipping a pooler — every raw connection eats roughly 10MB of server memory
- `VACUUM FULL` at peak hours, which takes an exclusive lock on the whole table

## Before you ship

- [ ] Indexes track real query patterns (cross-check `pg_stat_statements`)
- [ ] Composite indexes are ordered equality → sort → range
- [ ] `EXPLAIN ANALYZE` has been run on the critical queries
- [ ] Hot filtered subsets are backed by partial indexes
- [ ] A pooler (PgBouncer/pgcat) sits in front of Postgres
- [ ] Partitioning is on the table for anything past ~10M rows
- [ ] Unused indexes have been found and dropped
- [ ] `pg_stat_statements` is enabled for ongoing query monitoring
