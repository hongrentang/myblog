
---
title: "MySQL Replication Lag — How a Long-Running Query on the Replica Broke Read-After-Write Consistency"
date: 2026-06-07
weight: 100380
slug: "mysql-replication-lag-incident"
tags: ["mysql", "database", "troubleshooting", "replication", "performance"]
categories: ["Troubleshooting"]
description: "A MySQL replication lag incident — how a schema migration caused a long-running query on the replica, creating 30 minutes of replication delay and breaking read-after-write consistency for users"
keywords: "mysql replication lag, mysql master slave delay, seconds_behind_master, mysql replication troubleshooting, read after write consistency mysql"
draft: false
featured: true
cover:
  image: ""
  caption: "MySQL Replication Lag — Troubleshooting"
---
# MySQL Replication Lag — How a Long-Running Query on the Replica Broke Read-After-Write Consistency

## Common Search Queries

| English | Chinese |
|---------|---------|
| MySQL replication lag troubleshooting | MySQL 主从延迟排查 |
| Seconds_Behind_Master keeps increasing | Seconds_Behind_Master 持续增长 |
| Read-after-write consistency MySQL | MySQL 读写一致性 |
| MySQL slave lag cause | MySQL 从库延迟原因 |
| MySQL parallel replication setup | MySQL 并行复制配置 |
| pt-online-schema-change replication lag | pt-osc 主从延迟 |
| MySQL relay log space growing | MySQL relay log 暴涨 |
| Kill long running query on MySQL replica | 杀掉 MySQL 从库慢查询 |

---

## The Incident

### Environment

| Component | Specification |
|-----------|---------------|
| MySQL Version | 8.0.32 (Community) |
| Replication Mode | Asynchronous (Async) |
| Replication Format | Row-Based (RBR) |
| Topology | 1 Master + 1 Replica |
| App A | Writes to Master |
| App B | Reads from Replica |
| App C | Analytics queries on Replica |
| Table Size | ~10 million rows |
| Storage Engine | InnoDB |

### Timeline

| Time (UTC+8) | Event |
|--------------|-------|
| 14:00 | Peak business hours start |
| 14:15 | DBA runs `ALTER TABLE orders ADD COLUMN` on Master |
| 14:16 | Master write latency spikes; DDL blocks subsequent writes via MDL |
| 14:18 | Replica CPU reaches 100% |
| 14:20 | Customer support receives first reports of missing orders |
| 14:22 | On-call engineer is paged |
| 14:25 | `SHOW REPLICA STATUS` reveals `Seconds_Behind_Master` > 1800s |
| 14:28 | Root cause identified: analytical query on replica blocking SQL thread |
| 14:30 | Analytical query killed on replica |
| 14:32 | Replica starts catching up |
| 14:45 | Replication lag returns to 0; service fully recovered |

### Symptoms

- **Users reported that orders submitted successfully (via App A writing to Master) were invisible upon page refresh (App B reading from Replica).**
- Replica server CPU was pegged at 100%.
- Replica disk IO wait was elevated.
- Application error logs showed no database connection errors — queries returned successfully but with stale data.
- The replica's `Relay_Log_Space` had ballooned to several GB.

---

## Background

### MySQL Asynchronous Replication

In MySQL's default asynchronous replication, the Master writes events to its binary log (`binlog`), and the Replica pulls those events via the I/O thread into its relay log (`relay-log`). The SQL thread on the Replica then replays those events sequentially.

```
Master: [Transaction] → Binlog → Network → 
Replica I/O Thread: → Relay Log → 
Replica SQL Thread: → Storage Engine
```

The critical property of **asynchronous replication** is: the Master does **not wait** for the Replica to confirm that events have been applied. A transaction committed on the Master is immediately visible to Master readers, but may not appear on the Replica for some time. This is the fundamental source of read-after-write inconsistency.

### How `Seconds_Behind_Master` Works

`Seconds_Behind_Master` is the most commonly used metric for measuring replication lag, but it is frequently misunderstood:

- It is the difference between the timestamp on the event currently being executed by the SQL thread and the current system time on the replica.
- If the SQL thread is waiting (no events to apply), it reports `0`.
- If the SQL thread is actively applying events and the replica's system clock is ahead of the master's clock, the metric can be misleading.
- It does **not** measure the volume of pending events in the relay log.
- In MySQL 8.0, it can show `NULL` if the SQL thread is not running, or if `Replica_Delay` is configured.

### Binary Log Position vs. GTID

There are two ways to track replication position:

| Method | Description | Pros | Cons |
|--------|-------------|------|------|
| **Binary Log Position** | `(binlog_filename, offset)` pair. Each binlog event increments the offset. | Simple, widely understood | Fragile — if binlog files change on master, coordinates can be invalidated |
| **GTID** (Global Transaction ID) | `(source_id:transaction_id)` e.g., `a1b2c3d4-1234-4321-aaaa-bbbbbbbb:1-500`. Every transaction gets a unique ID. | Self-healing — replica can automatically skip already-applied transactions | Slightly more complex to set up |

In this incident, the environment used **position-based replication**.

---

## Investigation

All commands below were executed on the Replica unless otherwise noted. The investigation followed a systematic narrowing-down approach.

### Step 1: Check Replication Status — `SHOW REPLICA STATUS\G`

The first command any DBA runs when replication lag is suspected:

```sql
SHOW REPLICA STATUS\G
```

Key fields observed:

```
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
              Source_Log_File: mysql-bin.000147
          Read_Master_Log_Pos: 2891345678
               Relay_Log_File: relay-bin.000345
                Relay_Log_Pos: 451234567
        Relay_Source_Log_File: mysql-bin.000147
           Exec_Master_Log_Pos: 1098765432   ← Position of last applied event
               Relay_Log_Space: 4781234567   ← ~4.7 GB of pending relay log
           Seconds_Behind_Master: 1832       ← ~30 minutes behind!
```

The immediate red flags:
- `Seconds_Behind_Master: 1832` — the replica was over 30 minutes behind.
- `Relay_Log_Space: 4781234567` — nearly 4.7 GB of unapplied relay log events were queued.
- The difference between `Read_Master_Log_Pos` (2,891,345,678) and `Exec_Master_Log_Pos` (1,098,765,432) is **~1.79 GB** of binlog events waiting to be applied. This is the raw backlog volume.

### Step 2: Check Thread Status — `SHOW PROCESSLIST;`

To understand *why* the SQL thread is falling behind:

```sql
SHOW PROCESSLIST;
```

| Id | User | Host | db | Command | Time | State | Info |
|----|------|------|----|---------|------|-------|------|
| 10 | system user | | | Connect | 0 | Waiting for source to send event | |
| 11 | system user | | | Connect | 1832 | Applying batch of row changes (update) | |
| 12 | app_user | 10.0.0.5:43210 | orders | Query | **812** | **Sending data** | `SELECT ... FROM orders WHERE ... GROUP BY ... ORDER BY ...` |
| 13 | app_user | 10.0.0.6:43211 | orders | Query | 2 | Creating sort index | `SELECT ... LIMIT 10` |

Critical finding: **Thread 12** had been running for **812 seconds** (~13.5 minutes) and was in `Sending data` state — a heavy analytical query reading from the `orders` table. This query consumed nearly all CPU and IO resources on the replica, starving the SQL thread (Thread 11) of resources needed to apply binlog events.

### Step 3: Identify the Problematic Query

```sql
SELECT THREAD_ID, PROCESSLIST_ID, THREAD_OS_ID, 
       PROCESSLIST_INFO, PROCESSLIST_TIME, PROCESSLIST_STATE
FROM performance_schema.threads
WHERE PROCESSLIST_ID = 12\G
```

Checking the full query:

```sql
SHOW FULL PROCESSLIST;
```

The analytical query was:

```sql
SELECT user_id, COUNT(*) as order_count, SUM(amount) as total_spent
FROM orders
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
GROUP BY user_id
ORDER BY total_spent DESC
LIMIT 100;
```

This query scanned millions of rows, performed a full table scan or an inefficient index scan on `orders`, and consumed the replica's CPU for over 13 minutes.

### Step 4: Check Binlog Format

```sql
SHOW VARIABLES LIKE 'binlog_format';
```

```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
```

The environment used **Row-Based Replication (RBR)**. This is important because:

- With RBR, the `ALTER TABLE` on the master generates a row-based DDL event.
- More critically, every row-change event after the DDL must be replicated row-by-row.
- RBR events are generally larger than statement-based equivalents, making the relay log grow faster under backlog conditions.

### Step 5: Check SQL Thread State

```sql
SHOW SLAVE STATUS\G  -- MySQL < 8.0.22
-- or in MySQL 8.0.22+:
SHOW REPLICA STATUS\G
```

Look at `Slave_SQL_Running_State`:

```
Slave_SQL_Running_State: Applying batch of row changes (update)
```

This confirmed the SQL thread was actively working (not waiting), but was bottlenecked.

### Step 6: Master Side Check

On the Master, we checked:

```sql
SHOW MASTER STATUS\G
```

```
*************************** 1. row ***************************
             File: mysql-bin.000147
         Position: 2891345678
         Binlog_Do_DB: 
     Binlog_Ignore_DB: 
Executed_Gtid_Set: a1b2c3d4-1234-4321-aaaa-bbbbbbbbbbbb:1-5001234
```

The Master's binlog position (2,891,345,678) matched the `Read_Master_Log_Pos` on the replica, confirming the I/O thread was keeping up. The bottleneck was purely on the SQL thread side.

### Step 7: Calculate Approximate Lag

```sql
-- On Master
SHOW BINARY LOGS;
```

```
+--------------------+-----------+
| Log_name           | File_size |
+--------------------+-----------+
| mysql-bin.000145   | 1073741824|
| mysql-bin.000146   | 1073741824|
| mysql-bin.000147   | 2891345678|
+--------------------+-----------+
```

Each binlog file is 1 GB. Unapplied events span ~1.79 GB across binlog files. At typical replay speed on a loaded replica (~20-50 MB/s for RBR), this translated to **30+ minutes of backlog**.

---

## Root Cause

The incident resulted from a **chain of compounding factors**:

### 1. `ALTER TABLE` Trigger — Metadata Lock (MDL) Contention

When the DBA ran:

```sql
ALTER TABLE orders ADD COLUMN rebate_tier TINYINT DEFAULT 0;
```

MySQL's InnoDB requires a **metadata lock (MDL)** for DDL operations. The `ALTER TABLE` acquired an exclusive MDL on the `orders` table. This blocked:

- All subsequent `INSERT`, `UPDATE`, `DELETE` statements on `orders` (they need shared MDL).
- All subsequent `SELECT ... FOR UPDATE` statements.

On a busy OLTP system with ~10M rows, rebuilding the table (even with `ALGORITHM=INPLACE` or `INSTANT` if applicable) caused significant write queuing.

### 2. Write Queue on Master → Binlog Explosion

With writes queued behind the DDL, the master's binary log accumulated a large volume of batched row-change events. Once the DDL completed and the lock was released, all queued writes were committed in rapid succession, creating a dense burst of binlog events.

The replica's I/O thread dutifully pulled all these events into the relay log, but...

### 3. Heavy Analytical Query Starved the SQL Thread

The analytical query on Thread 12 (`SELECT ... GROUP BY ... ORDER BY ... LIMIT 100`) consumed nearly all available CPU on the replica (8 vCPU). The SQL thread, which is **single-threaded by default**, could not compete for resources. With the replica CPU at 100%, the SQL thread made negligible progress.

### 4. Single-Threaded SQL Thread (The Critical Bottleneck)

```sql
SHOW VARIABLES LIKE 'replica_parallel_workers';
```

```
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| replica_parallel_workers | 0   |
+------------------------+-------+
```

`replica_parallel_workers = 0` means **parallel replication was disabled**. The replica was replaying binlog events with a single SQL thread.

In MySQL 8.0, parallel replication (`replica_parallel_workers > 0`) allows the replica to apply transactions from different databases, or even the same database (with `LOGICAL_CLOCK`), in parallel. Without it, a single analytical query on the replica can bring the entire replication pipeline to a crawl.

### 5. Compounding `Seconds_Behind_Master`

As the replica fell further behind:
- The relay log grew (4.7 GB).
- Disk IO for relay log read/write competed with the analytical query's temp table writes.
- The longer the analytical query ran, the more events accumulated, making catch-up take even longer.

The sequence formed a **vicious cycle**:

```
DDL on Master → MDL blocks writes → Writes burst in binlog → 
Replica I/O thread pulls all events → Relay log grows → 
Analytical query consumes CPU → SQL thread starved → 
Lag increases → Relay log grows further → More to catch up
```

---

## Resolution

### Emergency Fix (Immediate)

**Step 1: Kill the long-running analytical query on the replica.**

```sql
-- Find the thread:
SHOW PROCESSLIST;

-- Kill the query (not the connection):
KILL QUERY 12;

-- If the query doesn't stop, kill the connection:
KILL 12;
```

After killing the query, the replica CPU dropped from 100% to ~30% almost immediately.

**Step 2: Reset replication to accelerate catch-up.**

```sql
STOP REPLICA;
START REPLICA;
```

This resets the I/O and SQL threads. Sometimes a fresh start clears internal buffer contention. In our case, after the kill, just restarting the replica threads was sufficient.

**Step 3: Monitor lag recovery.**

```sql
SHOW REPLICA STATUS\G
```

Observe `Seconds_Behind_Master` decreasing:

| Time | Seconds_Behind_Master |
|------|----------------------|
| 14:30 | 1832 |
| 14:33 | 1550 |
| 14:36 | 1120 |
| 14:39 | 680 |
| 14:42 | 210 |
| 14:45 | 0 |

**Step 4: Verify data consistency.**

```sql
-- Quick check on replica
SELECT COUNT(*) FROM orders WHERE rebate_tier = 0;

-- Compare master and replica checksums (sample)
CHECKSUM TABLE orders;
```

The replica took approximately **15 minutes** to catch up after the analytical query was killed and the replica threads were restarted.

### Long-Term Fixes

**1. Enable Parallel Replication**

```ini
# my.cnf on replica
replica_parallel_workers = 4
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = 1
```

- `replica_parallel_workers = 4`: Use 4 parallel worker threads for applying binlog events.
- `replica_parallel_type = LOGICAL_CLOCK`: Allow parallel applying of transactions that were committed on the master within the same logical clock window (i.e., transactions that did not conflict).
- `replica_preserve_commit_order = 1`: Ensure transactions commit in the same order on the replica as on the master, which is critical for consistency.

With parallel replication, the impact of a single analytical query on replication lag is significantly reduced because worker threads can make progress even if one worker is momentarily starved.

**2. Use pt-online-schema-change for Schema Migrations**

[pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html) (pt-osc) from Percona Toolkit avoids blocking writes during DDL:

```bash
pt-online-schema-change \
  h=localhost,D=production,t=orders \
  --alter "ADD COLUMN rebate_tier TINYINT DEFAULT 0" \
  --execute
```

How pt-osc works:
1. Creates a shadow table with the new schema.
2. Adds triggers to the original table to capture changes.
3. Copies data incrementally in chunks.
4. Swaps tables atomically via `RENAME TABLE`.

This approach holds the MDL lock only for a brief moment during the final table swap, reducing the write-blocking window from minutes to milliseconds.

**3. Separate Analytical Queries to a Dedicated Read Replica**

The analytical query that triggered this incident should never have been running on the same replica serving live user traffic.

Recommended architecture:

```
Master (writes)
 ├── Replica A (serves App B — live user reads)
 └── Replica B (serves App C — analytical/BI queries)
```

By dedicating a separate replica for analytical workloads, heavy `GROUP BY` and `ORDER BY` queries cannot interfere with replication or degrade user-facing read performance.

**4. Implement Replication Lag Monitoring and Alerting**

```sql
-- Monitor script (run every 30 seconds):
SELECT
  NOW() AS check_time,
  VARIABLE_VALUE AS seconds_behind_master
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Seconds_Behind_Master';
```

Or use a monitoring tool like Prometheus + mysqld_exporter with alert rules:

```yaml
# Alert rules example
groups:
  - name: mysql_replication
    rules:
      - alert: MySQLReplicationLagHigh
        expr: mysql_slave_status_seconds_behind_master > 300
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MySQL replication lag is {{ $value }}s"
```

Alert threshold suggestions:
- **Warning**: `Seconds_Behind_Master > 30` for 1 minute
- **Critical**: `Seconds_Behind_Master > 300` for 2 minutes

**5. Application-Level Read-After-Write Consistency**

For critical write-then-read workflows (like order submission), the application should read from Master after writing:

```python
# Before (might read stale data from replica):
def submit_order(user_id, items):
    order_id = db_master.execute("INSERT INTO orders ...")
    order = db_replica.query("SELECT * FROM orders WHERE id = ?", order_id)  # BUG!
    return order

# After (read from master for consistency):
def submit_order(user_id, items):
    order_id = db_master.execute("INSERT INTO orders ...")
    order = db_master.query("SELECT * FROM orders WHERE id = ?", order_id)  # OK
    return order

# Alternative: Read-after-write with delay tolerance:
def submit_order(user_id, items):
    order_id = db_master.execute("INSERT INTO orders ...")
    time.sleep(0.5)  # Wait briefly for replication
    order = db_replica.query("SELECT * FROM orders WHERE id = ?", order_id)
    return order
```

A more sophisticated approach is to use **session consistency** or **read-your-writes consistency** patterns:

```python
class ReadYourWriteRouter:
    def __init__(self):
        self.recent_writes = {}  # user_id -> timestamp

    def after_write(self, user_id):
        self.recent_writes[user_id] = time.time()

    def resolve_read_target(self, user_id, max_staleness=5.0):
        last_write = self.recent_writes.get(user_id, 0)
        if time.time() - last_write < max_staleness:
            return "master"
        return "replica"
```

---

## Lessons Learned

### Technical Lessons

| # | Lesson | Action Taken |
|---|--------|-------------|
| 1 | DDL operations on large tables block writes via MDL | Adopt pt-online-schema-change for all schema migrations |
| 2 | Default MySQL replication is single-threaded, making it fragile | Enable parallel replication on all replicas |
| 3 | Heavy analytical queries on replicas degrade replication speed | Separate OLTP and OLAP traffic at the replica level |
| 4 | `Seconds_Behind_Master` alone is insufficient for full monitoring | Add `Relay_Log_Space`, `Exec_Master_Log_Pos` gap, and replication throughput metrics |
| 5 | RBR generates larger binlog events than SBR, impacting catch-up speed under load | Account for binlog format when sizing replica capacity |
| 6 | Without monitoring, replication lag can go undetected for extended periods | Deploy lag alerting with graduated severity levels |

### Process Lessons

- **Change Management**: The `ALTER TABLE` was performed during peak hours without prior review. All schema changes should follow a change management process with planned execution windows (preferably off-peak).
- **Runbooks**: The team had no documented runbook for replication lag incidents. Post-incident, a runbook was created covering detection, investigation, and emergency response steps.
- **Testing**: The heavy analytical query had never been tested against production data volumes. Staging environments should mirror production data size for query performance testing.

---

## Summary

### Incident Timeline Summary

```
Time        Event
─────       ───────────────────────────────────────────────────────────
14:15       DBA runs ALTER TABLE orders ADD COLUMN on Master (10M rows)
14:16       MDL lock on Master blocks writes; write queue builds
14:18       Replica CPU 100% due to analytical query (running since 14:14)
14:20       Users report orders missing on page refresh
14:22       On-call engineer paged
14:25       SHOW REPLICA STATUS → Seconds_Behind_Master = 1832s
14:28       Root cause identified: analytical query starving SQL thread
14:30       KILL QUERY 12 on replica; CPU drops; restart replica threads
14:33       Lag begins decreasing (1550s → 1120s → 680s ...)
14:45       Lag reaches 0; service fully recovered
```

### Key Metrics at Peak

| Metric | Value |
|--------|-------|
| `Seconds_Behind_Master` | 1832s (~30 min) |
| `Relay_Log_Space` | ~4.7 GB |
| Binlog backlog volume | ~1.79 GB |
| Analytical query runtime | 812s before being killed |
| Replica CPU | 100% |
| `replica_parallel_workers` | 0 |
| Recovery time (kill → caught up) | ~15 min |

### Preventive Measures Summary

| Measure | Status | Owner |
|---------|--------|-------|
| Enable parallel replication (`replica_parallel_workers=4`) | Done | DBA |
| Adopt pt-online-schema-change for all DDL | Done | DBA Team |
| Add dedicated replica for analytical queries | In progress | Infrastructure |
| Deploy replication lag monitoring + alerting | Done | Observability |
| Implement read-your-writes pattern in order service | In progress | Backend Team |
| Create replication lag incident runbook | Done | SRE |
| Add ALTER TABLE review step to change management | Done | Engineering Manager |

The most important takeaway from this incident is that **replication lag is not just a database metric — it is a user-facing issue**. A 30-minute delay means 30 minutes of users seeing their orders as "missing," which erodes trust and directly impacts business metrics. Every team running MySQL in a master-replica topology should treat replication lag monitoring with the same urgency as uptime monitoring.

---

*Written in 2026. MySQL 8.0.32. This article is part of the production incident troubleshooting series at [blog.777157.xyz](https://blog.777157.xyz).*
