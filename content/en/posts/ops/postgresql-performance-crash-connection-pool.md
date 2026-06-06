---
title: "PostgreSQL Performance Crash — How a Missing Index Brought the Database to Its Knees"
date: 2026-06-06
weight: 100360
slug: "postgresql-performance-crash-connection-pool"
tags: ["postgresql", "database", "troubleshooting", "performance", "connection"]
categories: ["Troubleshooting"]
description: "A PostgreSQL performance incident — how a missing index caused full table scans, maxed out connections, and took down the production database, with a step-by-step investigation using pg_stat_activity and pg_stat_statements"
keywords: "postgresql performance troubleshooting, postgres connection pool full, pg_stat_activity, pg_stat_statements, missing index postgresql"
draft: false
featured: true
cover:
  image: ""
  caption: "PostgreSQL Performance Crash — Troubleshooting"
---

## Common Search Queries

- PostgreSQL connection pool exhausted
- PostgreSQL max_connections reached
- PostgreSQL performance suddenly slow
- PostgreSQL pg_stat_activity all active
- How to find slow queries in PostgreSQL
- PostgreSQL missing index causing full table scan
- pg_stat_statements query performance analysis
- PostgreSQL connection storm thundering herd
- PostgreSQL kill long running queries
- PostgreSQL create index concurrently

---

## The Incident

**Environment:**
| Component | Spec |
|-----------|------|
| PostgreSQL Version | 15.4 |
| RAM | 16 GB |
| vCPU | 4 cores |
| Storage | 500 GB SSD |
| max_connections | 200 |
| OS | Ubuntu 22.04 LTS |

It was a typical Tuesday morning. At 10:00 AM sharp — the daily peak hour — the monitoring system began firing alerts in rapid succession.

**Timeline:**

| Time | Event |
|------|-------|
| 10:01 | Prometheus alert: PostgreSQL connections at 95% (190/200) |
| 10:02 | API response time spikes from 50ms to 15s |
| 10:03 | Application servers return HTTP 500 errors |
| 10:04 | DB CPU usage hits 100%, iowait at 60% |
| 10:05 | `FATAL: remaining connection slots are reserved for non-replication superuser connections` |
| 10:06 | Pager duty alert triggered — incident response begins |

By 10:05, the production database was effectively offline. Every application server in the cluster was unable to acquire a database connection. The few connections that succeeded took 20-30 seconds per query — a 400x-600x degradation from the normal 50ms response time.

---

## Background

Two days before the incident, the product team shipped a new feature: **Order Dashboard v2**. This feature introduced a new API endpoint `/api/v2/orders/summary` that allowed operations staff to query order summaries by user ID, status, and date range.

The query looked like this:

```sql
SELECT user_id, status, count(*) as order_count, sum(amount) as total_amount
FROM orders
WHERE user_id = 12345
  AND status IN ('pending', 'processing', 'shipped')
  AND created_at >= '2026-06-01'
GROUP BY user_id, status
ORDER BY status;
```

The ORM (SQLAlchemy) generated this query from a high-level model query. During the code review, the team focused on business logic correctness — status transitions, amount calculations, pagination. Nobody checked the query plan.

The `orders` table had **5 million rows**. There were indexes on `user_id` alone and `created_at` alone, but **no composite index** covering all three filter columns together. PostgreSQL's query planner, seeing no efficient path, chose a **sequential scan** on the entire 5M-row table.

In staging, this wasn't caught because:
- Staging had only 50K rows (not 5M)
- Staging had no concurrent load
- Staging queries completed in 50ms, masking the problem

---

## Investigation

### Step 1: Check Connection Count

The first sign of trouble was obvious — the app couldn't connect. We logged into the database directly:

```sql
SELECT count(*) FROM pg_stat_activity;
```

```
 count
-------
   200
(1 row)
```

All 200 connection slots were consumed. Let's see what state they're in:

```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```

```
 state  | count
--------+-------
 active |   180
 idle   |    20
(2 rows)
```

180 connections in `active` state — that means 180 queries were actively running (or blocking). Normally we see ~20 active and the rest idle in a connection pool.

### Step 2: Find the Long-Running Queries

```sql
SELECT pid,
       now() - pg_stat_activity.query_start AS duration,
       left(query, 120) AS query_preview,
       state,
       wait_event_type,
       wait_event
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC
LIMIT 10;
```

```
 pid  | duration | query_preview                                        | state  | wait_event_type | wait_event
------+----------+------------------------------------------------------+--------+-----------------+------------
 1234 | 00:08:12 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1235 | 00:07:55 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1236 | 00:07:41 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1267 | 00:07:30 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1278 | 00:07:12 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1301 | 00:06:55 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1312 | 00:06:40 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1334 | 00:06:22 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1356 | 00:05:58 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
 1378 | 00:05:30 | SELECT user_id, status, count(*) as order_count...   | active | IO              | DataFileRead
```

The pattern was unmistakable — the same query, running across dozens of connections, each taking 5-8 minutes. The wait event `DataFileRead` indicated heavy I/O from reading table blocks into shared buffers — a classic sign of sequential scans on large tables.

### Step 3: Inspect Query Performance with pg_stat_statements

We checked the `pg_stat_statements` view to understand cumulative query performance:

```sql
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time,
       rows,
       shared_blks_hit,
       shared_blks_read,
       shared_blks_hit::numeric / (shared_blks_hit + shared_blks_read)::numeric AS hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

```
query                                                                           | calls | total_exec_time | mean_exec_time | rows  | shared_blks_hit | shared_blks_read | hit_ratio
-------------------------------------------------------------------------------+-------+-----------------+----------------+-------+-----------------+------------------+-----------
SELECT user_id, status, count(*), sum(amount) FROM orders WHERE user_id...     | 16854 | 28341920.5      | 1682.1         | 50562 | 512384          | 48921568         | 0.010
SELECT * FROM orders WHERE order_id = $1                                        | 98523 | 12453.8         | 0.126          | 98523 | 885234          | 1234             | 0.999
UPDATE orders SET status = $1 WHERE order_id = $2                               | 45213 | 8923.4          | 0.197          | 45213 | 452389          | 892              | 0.998
INSERT INTO orders (user_id, amount, status, created_at) VALUES ($1,$2,$3,$4)   | 32145 | 4567.2          | 0.142          | 32145 | 321456          | 456              | 0.999
```

**The findings were damning:**

- The bad query accounted for **28,341,920 ms** of total execution time (nearly 8 hours of cumulative CPU time)
- Mean execution time: **1,682 ms** (1.7 seconds per query)
- **Hit ratio: 0.01** — only 1% of blocks were found in shared buffers, meaning 99% of reads came from disk
- In contrast, all other queries had hit ratios of 0.998+ and mean times under 200ms
- The query had been called **16,854 times** since the new deployment

### Step 4: EXPLAIN ANALYZE the Slow Query

We grabbed the exact query text and ran `EXPLAIN (ANALYZE, BUFFERS)`:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, status, count(*) as order_count, sum(amount) as total_amount
FROM orders
WHERE user_id = 12345
  AND status IN ('pending', 'processing', 'shipped')
  AND created_at >= '2026-06-01'
GROUP BY user_id, status
ORDER BY status;
```

```
                                                           QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=285412.45..285412.48 rows=12 width=48) (actual time=5842.3..5842.3 rows=3 loops=1)
   Sort Key: status
   Sort Method: quicksort  Memory: 25kB
   ->  Finalize GroupAggregate  (cost=285410.20..285412.18 rows=12 width=48) (actual time=5842.2..5842.2 rows=3 loops=1)
         Group Key: user_id, status
         ->  Gather  (cost=285409.78..285411.94 rows=24 width=48) (actual time=5842.1..5850.4 rows=9 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               ->  Partial GroupAggregate  (cost=284409.78..284411.94 rows=12 width=48) (actual time=5826.5..5826.6 rows=3 loops=3)
                     Group Key: user_id, status
                     ->  Sort  (cost=284409.78..284409.81 rows=12 width=40) (actual time=5826.4..5826.4 rows=8 loops=3)
                           Sort Key: user_id, status
                           Sort Method: quicksort  Memory: 25kB
                           ->  Seq Scan on orders  (cost=0.00..284409.50 rows=12 width=40) (actual time=5623.8..5826.2 rows=8 loops=3)
                                 Filter: ((created_at >= '2026-06-01'::date) AND (user_id = 12345) AND (status = ANY ('{pending,processing,shipped}'::text[])))
                                 Rows Removed by Filter: 5000000
                                 Buffers: shared hit=384 read=49512 dirtied=152
 Planning:
   Buffers: shared hit=36 read=12 dirtied=8
 Planning Time: 0.348 ms
 Execution Time: 5850.58 ms
```

**Critical observations from the plan:**

1. **Seq Scan on orders** — PostgreSQL scanned **all 5 million rows** of the orders table
2. **Rows Removed by Filter: 5,000,000** — For each query, 5M rows were read and discarded
3. **Execution Time: 5,850 ms** — Nearly 6 seconds per query
4. **Buffers: shared hit=384 read=49512** — 49,512 blocks read from disk (about 387 MB per query)
5. The query only returned **3 rows** — a needle in a haystack

### Step 5: Check Existing Indexes

```sql
SELECT schemaname,
       tablename,
       indexname,
       indexdef
FROM pg_indexes
WHERE tablename = 'orders'
ORDER BY indexname;
```

```
 schemaname | tablename | indexname              | indexdef
------------+-----------+------------------------+----------------------------------------------------------
 public     | orders    | idx_orders_created_at  | CREATE INDEX idx_orders_created_at ON orders USING btree (created_at)
 public     | orders    | idx_orders_order_id    | CREATE INDEX idx_orders_order_id ON orders USING btree (order_id)
 public     | orders    | idx_orders_user_id     | CREATE INDEX idx_orders_user_id ON orders USING btree (user_id)
```

There were indexes on `user_id` and `created_at` individually. However, PostgreSQL's query planner cannot efficiently combine two separate B-tree indexes in this scenario — it needs to filter on `user_id`, `status`, AND `created_at` simultaneously. Without a **composite index** covering all three columns, the planner fell back to a sequential scan, correctly calculating that the combined selectivity of the two separate indexes wasn't worth the overhead.

**Why separate indexes aren't enough:** PostgreSQL can do a Bitmap Scan combining two indexes (bitmap-and), but with `status` as an additional filter and the `GROUP BY`/`ORDER BY` requirements, the optimizer concluded that reading the entire table was cheaper than the bitmap-and approach. For a table with 5M rows, this decision was catastrophic.

---

## Root Cause

1. **No composite index on (user_id, status, created_at)** — The new query filtered on all three columns, but only individual indexes existed. This forced a full table scan on 5M rows per query.

2. **Full table scan on every execution** — Each query read ~387 MB from disk, taking 5-8 seconds. With 50+ concurrent requests, the I/O subsystem was saturated.

3. **Connection accumulation** — As queries took longer (5-8s each), connections were held open longer. With 180 active connections all waiting on I/O, the connection pool was exhausted.

4. **max_connections=200 ceiling** — Once all 200 slots were filled, new connections were rejected with the fatal error. Application servers logging the error immediately retried, creating a **thundering herd** effect that made recovery impossible without manual intervention.

5. **No query review in CI/CD** — The deployment pipeline had no step to analyze query plans or detect sequential scans on large tables. The staging environment's small dataset masked the issue entirely.

---

## Resolution

### Emergency Triage (10:06 - 10:12)

**Step 1: Kill the long-running queries to free connections.**

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid IN (
    SELECT pid
    FROM pg_stat_activity
    WHERE state = 'active'
      AND query ILIKE '%orders%'
      AND now() - query_start > interval '30 seconds'
);
```

This killed 178 queries immediately. Connections dropped from 200 to 22.

**Step 2: Temporarily increase max_connections to handle the surge.**

```sql
ALTER SYSTEM SET max_connections = 300;
```

Then restart the PostgreSQL service (or use `pg_ctl reload` — though `max_connections` requires a restart):

```bash
sudo systemctl restart postgresql
```

> **Note:** Changing `max_connections` requires a full restart, which causes a brief downtime. In our case, the database was already unusable, so the 30-second restart was acceptable. For future incidents, consider using PgBouncer to multiplex connections instead.

**Step 3: Verify the database is responsive again.**

```sql
SELECT count(*) FROM pg_stat_activity;
```

```
 count
-------
    18
(1 row)
```

API response times returned to normal (50ms).

### Permanent Fix (10:30 - 10:45)

**Create the missing composite index.**

We used `CREATE INDEX CONCURRENTLY` to avoid locking the table for writes:

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_status_created
ON orders (user_id, status, created_at);
```

This took about 4 minutes on the 5M-row table (non-blocking).

**Verify the query plan after the index:**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, status, count(*) as order_count, sum(amount) as total_amount
FROM orders
WHERE user_id = 12345
  AND status IN ('pending', 'processing', 'shipped')
  AND created_at >= '2026-06-01'
GROUP BY user_id, status
ORDER BY status;
```

```
                                                                  QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------------
 Sort  (cost=12.34..12.37 rows=12 width=48) (actual time=0.045..0.046 rows=3 loops=1)
   Sort Key: status
   Sort Method: quicksort  Memory: 25kB
   ->  GroupAggregate  (cost=12.04..12.16 rows=12 width=48) (actual time=0.032..0.036 rows=3 loops=1)
         Group Key: user_id, status
         ->  Index Only Scan using idx_orders_user_status_created on orders  (cost=0.04..12.06 rows=12 width=40) (actual time=0.020..0.025 rows=8 loops=1)
               Index Cond: ((user_id = 12345) AND (status = ANY ('{pending,processing,shipped}'::text[])) AND (created_at >= '2026-06-01'::date))
               Heap Fetches: 0
               Buffers: shared hit=6
 Planning:
   Buffers: shared hit=6
 Planning Time: 0.234 ms
 Execution Time: 0.082 ms
```

**Before vs. After comparison:**

| Metric | Before (Seq Scan) | After (Index Scan) | Improvement |
|--------|-------------------|--------------------|-------------|
| Execution Time | 5,850 ms | 0.082 ms | **71,000x faster** |
| Buffers Read | 49,512 from disk | 6 from shared buffers | **8,252x fewer I/O** |
| Rows Scanned | 5,000,000 | 8 | **625,000x fewer rows** |
| Hit Ratio | 0.01 (1%) | 1.0 (100%) | **Perfect** |
| Query Type | Seq Scan | Index Only Scan | — |

### Application-Level Changes

After the database was stabilized, we made several application-level fixes:

1. **Reduce connection pool size** — Each application instance was configured to use 20 connections. With 10 instances, that's 200 connections — exactly the limit. We reduced this to 5 per instance with PgBouncer in transaction mode:

   ```python
   # SQLAlchemy config before
   engine = create_engine(DB_URL, pool_size=20, max_overflow=10)
   
   # SQLAlchemy config after (with PgBouncer)
   engine = create_engine(DB_URL, pool_size=5, max_overflow=2)
   ```

2. **Add retry with exponential backoff** — Application code was updated to retry failed connections with jitter:

   ```python
   import time
   import random
   
   def execute_with_retry(query, max_retries=3):
       for attempt in range(max_retries):
           try:
               return execute(query)
           except OperationalError as e:
               if "remaining connection slots" in str(e):
                   wait = (2 ** attempt) + random.uniform(0, 1)
                   time.sleep(wait)
               else:
                   raise
       raise Exception("Max retries exceeded")
   ```

3. **Add query timeout** — A safety net to prevent any future runaway queries:

   ```sql
   ALTER DATABASE mydb SET statement_timeout = '30s';
   ```

4. **CI/CD query review** — We added an automated step to the CI pipeline that runs `EXPLAIN` on all new queries against a production-sized dataset and fails the build if any sequential scan is detected on tables larger than 100K rows. Tools like `pg_qualstats` and `auto_explain` can also help detect problematic queries in staging.

---

## Lessons Learned

### What Went Wrong

1. **No database review for new queries** — The code review process checked business logic but not query performance. A simple `EXPLAIN` would have caught the problem.
2. **Staging data mismatch** — Staging had 50K rows vs. production's 5M. The query performed fine in staging, giving false confidence.
3. **No query performance baseline** — We had monitoring for system metrics (CPU, memory, connections) but no query-level performance monitoring. `pg_stat_statements` was enabled but nobody looked at it proactively.
4. **No connection pooling** — Applications connected directly to PostgreSQL without PgBouncer or similar middleware, making connection exhaustion a single-point-of-failure.
5. **No statement timeout** — Without `statement_timeout`, runaway queries could run indefinitely, holding connections hostage.

### What We Improved

| Area | Before | After |
|------|--------|-------|
| Query Review | Manual code review only | Automated EXPLAIN in CI/CD |
| Connection Mgmt | Direct pooling per app | PgBouncer transaction pooling |
| Monitoring | System metrics only | pg_stat_statements dashboard + query performance alerts |
| Staging Data | 50K rows | Anonymized production snapshot (5M rows) |
| Query Safety | No timeout | statement_timeout = 30s |
| Incident Response | No runbook | Database incident runbook created |
| Capacity Planning | No connection sizing | Connection budget per service documented |

### Detection vs. Prevention

The most valuable lesson: **we detected the symptom (connections full) but not the cause (slow queries)**. If we had monitoring on query execution time and sequential scan detection, we would have caught this within minutes of deployment — not 2 days later when it brought down production.

---

## Summary

This incident is a textbook case of how a **missing index** can cascade into a full production outage:

```
New query without composite index
         │
         ▼
  Sequential scan on 5M rows
         │
         ▼
  Each query takes 5-8 seconds
         │
         ▼
  50 concurrent → connections held longer
         │
         ▼
  max_connections=200 exhausted
         │
         ▼
  New connections rejected →
  Apps retry → Thundering herd
         │
         ▼
    DATABASE OFFLINE ❌
```

The fix was straightforward — a composite index — but the impact was a full production outage lasting 45 minutes. The key takeaways:

- **Always `EXPLAIN` new queries before deploying to production**, especially those touching large tables
- **Use `pg_stat_statements`** as an early warning system for query performance regression
- **Deploy PgBouncer** to decouple application connections from database connections
- **Set `statement_timeout`** as a safety net
- **Match staging data volume** to production (or use realistic datasets)
- **Monitor query performance**, not just system metrics

The composite index reduced query time from 5,850 ms to 0.082 ms — a **71,000x improvement**. PostgreSQL is a powerful database, but it cannot overcome a missing index through optimization alone. The query planner is only as good as the indexes you give it.

---

## References

- [PostgreSQL Documentation: Using EXPLAIN](https://www.postgresql.org/docs/15/using-explain.html)
- [PostgreSQL Documentation: pg_stat_statements](https://www.postgresql.org/docs/15/pgstatstatements.html)
- [PostgreSQL Documentation: CREATE INDEX CONCURRENTLY](https://www.postgresql.org/docs/15/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)
- [PostgreSQL Documentation: Connection Settings](https://www.postgresql.org/docs/15/runtime-config-connection.html)
- [PostgreSQL Wiki: Index Maintenance](https://wiki.postgresql.org/wiki/Index_Maintenance)
- [PgBouncer Documentation](https://www.pgbouncer.org/)
