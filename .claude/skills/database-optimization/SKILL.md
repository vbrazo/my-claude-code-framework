---
name: database-optimization
description: Tuning database performance — query plans, index choices, N+1 detection, pooling, replicas, and partitioning for PostgreSQL and MySQL
---

# Database performance tuning

## Start with EXPLAIN

Never optimize blind — run `EXPLAIN ANALYZE` first and read the plan from the bottom up.

```sql
-- PostgreSQL
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT ...;

-- MySQL
EXPLAIN ANALYZE SELECT ...;
```

What the plan is telling you:
- **Seq Scan** on a large table → an index is missing
- **Nested Loop** with a high row count → a hash or merge join may be better
- **Sort** with no supporting index → index the sort column
- **Estimated vs actual rows** drifting apart → stale statistics; run `ANALYZE`

## Choosing indexes

### B-tree — the default, fits most cases
```sql
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_orders_user_date ON orders (user_id, created_at DESC);
```
Good for equality, ranges, and sorting. In a composite index the column order matters — lead with equality columns, then range and sort columns.

### Partial index (PostgreSQL)
```sql
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';
```
Worth it when a query always filters on the same condition — the index stays far smaller than a full one.

### GIN (PostgreSQL — arrays, JSONB, full-text)
```sql
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_docs_search ON documents USING GIN (to_tsvector('english', content));
```

### GiST (PostgreSQL — spatial and range types)
```sql
CREATE INDEX idx_locations_point ON locations USING GiST (coordinates);
CREATE INDEX idx_events_period ON events USING GiST (tsrange(start_at, end_at));
```

### Covering index — for index-only scans
```sql
-- PostgreSQL
CREATE INDEX idx_users_email_name ON users (email) INCLUDE (name);

-- MySQL
CREATE INDEX idx_users_email_name ON users (email, name);
```

## Catching N+1 queries

The tell: one query for the parents, then another query for each child.

```python
# BAD: N+1
users = db.query(User).all()
for user in users:
    print(user.orders)  # triggers query per user

# GOOD: eager load
users = db.query(User).options(joinedload(User.orders)).all()
```

```javascript
// BAD: N+1
const users = await User.findAll();
for (const user of users) {
  const orders = await Order.findAll({ where: { userId: user.id } });
}

// GOOD: batch load
const users = await User.findAll({ include: [Order] });
```

To spot it, turn on query logging and count queries per request — more than ten on a single endpoint is a red flag.

## Pooling connections

```
Rule of thumb: pool_size = (core_count * 2) + disk_count
Typical web app: 10-20 connections per app instance
```

PostgreSQL:
- Use PgBouncer in transaction mode for serverless/high-connection scenarios
- Set `idle_in_transaction_session_timeout = '30s'`
- Monitor with `pg_stat_activity`

MySQL:
- Set `max_connections` based on available RAM (each connection uses ~10MB)
- Use ProxySQL for connection multiplexing
- Monitor with `SHOW PROCESSLIST`

## Read replicas

- Send `SELECT`s to the replicas
- Send every write to the primary
- Budget for replication lag (usually 10–100ms)
- Don't read-after-write off a replica — go to the primary when consistency matters
- Route at the connection level, not per query

```python
# SQLAlchemy read replica routing
class RoutingSession(Session):
    def get_bind(self, mapper=None, clause=None):
        if self._flushing or self.is_modified():
            return engines["primary"]
        return engines["replica"]
```

## Partitioning strategies

### Range partitioning — time-series data
```sql
-- PostgreSQL
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY,
    created_at timestamptz NOT NULL,
    data jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_q1 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
CREATE TABLE events_2025_q2 PARTITION OF events
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');
```

### Hash partitioning — even distribution
```sql
CREATE TABLE sessions (
    id uuid PRIMARY KEY,
    user_id bigint NOT NULL
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

Reach for partitioning once a table passes 50–100GB, or when you need to drop old data fast.

## Optimization checklist

1. Run `EXPLAIN ANALYZE` and actually read the plan
2. Hunt for sequential scans on anything past ~10K rows
3. Confirm indexes are used (`idx_scan` in `pg_stat_user_indexes`)
4. Watch for implicit type casts that quietly disable an index
5. Swap `SELECT *` for the columns you need
6. Add `LIMIT` where you only need a slice
7. Prefer `EXISTS` over `COUNT(*) > 0`
8. Batch `INSERT`/`UPDATE` work (500–1000 rows at a time)
9. Keep functions off indexed columns in `WHERE`
10. Watch the slow-query log (pg: `log_min_duration_statement = 100`)

## Patterns that bite

- `LIKE '%term%'` on an unindexed column — use full-text search instead
- `ORDER BY RANDOM()` — sample with `TABLESAMPLE` or randomize in the app
- `SELECT DISTINCT` papering over a broken join
- A missing `WHERE` on `UPDATE`/`DELETE` — sanity-check with a `SELECT` first
- Long transactions sitting on locks
- `OFFSET` for deep pagination — switch to keyset/cursor paging
