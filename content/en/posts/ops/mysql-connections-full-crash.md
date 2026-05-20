---
title: "MySQL Connections Exhausted — From max_connections to Connection Pool Meltdown"
date: 2026-05-21
weight: 100110
slug: "mysql-connections-full-crash"
tags: ["mysql", "database", "troubleshooting"]
categories: ["MySQL"]
description: "Full postmortem of MySQL connection exhaustion — from misdiagnosing max_connections to finding slow query avalanche and connection leaks"
keywords: "mysql too many connections, mysql max_connections, mysql connection pool, connection leak, wait_timeout"
draft: false
featured: true
cover:
  image: "/images/mysql-connections-banner.svg"
  caption: "MySQL Connections Exhausted Troubleshooting"
---

# MySQL Connections Exhausted — From max_connections to Connection Pool Meltdown

## Symptoms

Wednesday afternoon — core business APIs all returning 500 errors.

From application logs:

```
2026-05-21 14:30:01.123 ERROR [http-nio-8080-exec-12] c.e.demo.controller.OrderController - Create order failed
com.zaxxer.hikari.pool.HikariPool$TimeoutException: HikariPool-1 - Connection is not available, request timed out after 30000ms
```

The app's connection pool couldn't get a database connection.

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"
```

```
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
```

```bash
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
```

```
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_connected | 151   |
```

All 151 connections used up.

**Impact**: Every service depending on this database went down — orders, payments, user services.

## Investigation

### Wrong Turn 1: Increased max_connections

First instinct: "The limit is too low, just bump it up."

```bash
mysql -u root -p -e "SET GLOBAL max_connections = 500;"
```

It worked briefly. 20 minutes later, connections hit 480.

```mysql
SELECT COUNT(*) FROM information_schema.processlist;
```

```
+----------+
| COUNT(*) |
+----------+
|      480 |
```

At this rate, 500 would be full soon enough.

**Lesson**: Raising `max_connections` just postpones the crash. Every connection consumes 2-8 MB of RAM — set it too high and you'll trigger an OOM instead.

### Wrong Turn 2: Killing Connections

```mysql
-- Find slow queries
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

```
+------+------+-----------+--------+---------+------+-----------+--------------------------------------------+
| id   | user | host      | db     | command | time | state     | info                                       |
+------+------+-----------+--------+---------+------+-----------+--------------------------------------------+
| 123  | app  | 10.0.0.1 | mydb   | Query   | 2345 | Sending data | SELECT * FROM orders WHERE status = 0 ... |
```

Some queries had been running for 2000+ seconds.

```mysql
KILL 123;
KILL 456;
```

Connections dropped briefly, then came right back — the app retried, pushing new slow queries in.

```mysql
-- Mass kill (my worst idea that day)
SELECT CONCAT('KILL ', id, ';') FROM information_schema.processlist
WHERE user != 'root' AND command != 'Sleep';
```

This killed normal requests too. More errors for the frontend.

**Lesson**: `KILL` is a symptom fix. Fix the queries, or the connections come back. Mass killing breaks legitimate traffic.

### Wrong Turn 3: Restarting MySQL

"Let's just restart MySQL." Worst decision of the day.

```bash
systemctl restart mysqld
```

15 minutes later, connections hit 200+ again. And the restart introduced two problems:

1. **Active connections dropped** — inflight queries failed, apps errored
2. **Cold Buffer Pool** — all cached data lost, massive disk IO for warmup

```bash
iostat -x 1 3
```

```
Device   r/s     w/s     rkB/s     wkB/s  await  %util
sda      2345.0  567.0   189000.0  45000.0  45.0  99.8
```

Disk at 99.8% util — Buffer Pool reloading from disk.

**Lesson**: MySQL restart is a last resort, not a routine recovery step. Cold Buffer Pool causes crippling performance degradation that makes everything worse.

### The Real Discovery: Root Cause

```mysql
SELECT
    user, host, db, command,
    COUNT(*) as count,
    ROUND(AVG(TIME)) as avg_time_seconds
FROM information_schema.processlist
GROUP BY user, host, db, command
ORDER BY count DESC;
```

```
+------+-----------+--------+---------+------+-----------------+
| user | host      | db     | command | count| avg_time_seconds |
+------+-----------+--------+---------+------+-----------------+
| app  | 10.0.1.1  | mydb   | Query   | 45   | 120             |
```

40+ active queries averaging 120 seconds each. Orders queries should return in under 100ms.

```bash
mysqldumpslow -s t -t 20 /var/log/mysql/mysql-slow.log
```

```
Count: 2345  Time=45.23s
  SELECT * FROM orders WHERE status = N ORDER BY create_time DESC
```

```mysql
SHOW INDEX FROM orders;
```

```
+--------+------------+----------+--------------+-------------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name |
+--------+------------+----------+--------------+-------------+
| orders |          0 | PRIMARY  |            1 | id          |
```

No index on `(status, create_time)`. Every query scans 12M rows. Each one takes 45 seconds.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct cause | Slow queries hold connections for 45+ seconds, exhausting the pool |
| Config flaw | `max_connections` at default 151, `wait_timeout` at 28800 (8h) |
| Root cause | Missing `(status, create_time)` index — full table scans on a 12M-row table |

Chain reaction: slow queries → connections held long → pool full → new requests blocked → app retries → more connections → avalanche.

## Solutions

### Option A: Add the Missing Index (Root Fix)

```mysql
ALTER TABLE orders ADD INDEX idx_status_create_time (status, create_time);
```

After index: query drops from 45s to 30ms.

### Option B: Tune Connection Timeouts

```mysql
SET GLOBAL wait_timeout = 300;
SET GLOBAL interactive_timeout = 300;
```

Disconnects idle Sleep connections after 5 minutes.

### Option C: Set Realistic max_connections

```mysql
SET GLOBAL max_connections = 200;
```

Match this to your app's actual pool size × number of instances.

**Option comparison**:

| Option | Scenario | Effect | Risk |
|--------|----------|--------|------|
| A. Add index | Slow queries filling pool | ✅ Root fix | Brief table lock during ALTER |
| B. Tune timeouts | Too many Sleep connections | ✅ Frees idle ones | Long transactions may drop |
| C. Raise max_connections | Underestimated pool size | ⚠️ Temporary | Too high triggers OOM |

### Verify Recovery

```bash
# 1. Check connection count
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
# Expected: 50-80

# 2. Check slow query log
tail -f /var/log/mysql/mysql-slow.log
# Expected: no more orders full table scans

# 3. Verify business API
curl -I https://api.example.com/orders
# Expected: 200 OK
```

✅ **Recovery verification**:
- `Threads_connected` stabilizes at 50-80
- Slow query log shows orders queries under 100ms
- Business APIs return 200 without connection timeouts

## Long-Term Prevention

```bash
# 1. Monitor connection utilization
# Threads_connected / max_connections > 0.8 → alert

# 2. Monitor slow queries
# long_query_time = 1, alert on any slow query

# 3. Regular index review
# Quarterly review of slow query log, identify missing indexes

# 4. Connection pool sizing
# HikariCP maximumPoolSize × app instances + buffer ≤ MySQL max_connections
```

## What I Learned

1. **Raising `max_connections` doesn't fix the problem — it just postpones the crash.** Every connection costs 2-8 MB of RAM. Blindly increasing it trades one failure mode (connection limit) for another (MySQL OOM).

2. **Don't restart MySQL to clear connections.** Cold Buffer Pool causes crippling disk IO that makes the slowdown worse. A restart should be the absolute last resort.

3. **Killing connections is a symptom fix.** Slow queries are the real problem — kill them and they come back. `EXPLAIN` and `SHOW INDEX` are the actual debugging tools for this class of issue.

4. **One missing index can bring down your entire database.** A single 45-second full table scan multiplied by 50 concurrent requests = 50 connections held hostage. That's how a slow query becomes a site outage.

5. **Connection pool sizing isn't a guessing game.** Formula: MySQL `max_connections` ≥ (all app instances × their `maximumPoolSize`) × 1.2 buffer. Before the index: 150+ connections. After the index: 50 was enough. The gap is entirely slow queries.
