---
title: "MySQL from Beginner to Pro · Part 14: Performance Monitoring and Slow Query Optimization"
date: 2025-01-18
weight: 1314
draft: false
tags: ["mysql"]
---

## Performance Optimization Overview

```
Identify problem → Analyze cause → Develop solution → Implement optimization → Verify results
```

MySQL performance optimization typically involves three levels:

1. **Query Level**: Are SQL statements efficient?
2. **Index Level**: Is index design reasonable?
3. **Configuration Level**: Are MySQL parameters adapted to hardware?

---

## 1. Slow Query Log

The slow query log is the **first entry point** for performance optimization.

### Enabling the Slow Query Log

```sql
-- View current slow query configuration
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- Enable (temporary)
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;        -- Log queries exceeding 1 second
SET GLOBAL log_queries_not_using_indexes = ON;  -- Log queries not using indexes

-- Enable permanently (my.cnf)
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON
```

### Viewing Slow Queries

```sql
-- View slow query count
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- View current slow queries
SELECT COUNT(*) FROM mysql.slow_log;
```

### mysqldumpslow Analysis

```bash
# Sort by execution count
mysqldumpslow -s c /var/log/mysql/slow.log

# Sort by average query time (most common)
mysqldumpslow -s t /var/log/mysql/slow.log

# View only top 10
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# Sort by lock wait time
mysqldumpslow -s l /var/log/mysql/slow.log

# Output detailed information
mysqldumpslow -a -s t -t 5 /var/log/mysql/slow.log
```

### pt-query-digest (Percona Toolkit)

```bash
# Install
sudo apt install percona-toolkit

# Analyze slow queries
pt-query-digest /var/log/mysql/slow.log

# Output results to file
pt-query-digest /var/log/mysql/slow.log > slow_query_report.txt

# Analyze real-time MySQL queries
pt-query-digest --processlist mysql://user@localhost
```

---

## 2. EXPLAIN Execution Plan

EXPLAIN is the go-to tool for analyzing individual queries.

### Basic Usage

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';

-- MySQL 8.0.16+ supports EXPLAIN ANALYZE (outputs actual execution time)
EXPLAIN ANALYZE SELECT * FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.email = 'test@example.com';
```

### Output Interpretation

| Column | Meaning | Good | Bad |
|----|------|----|----|
| `id` | SELECT sequence number in query | Simple query id = 1 | — |
| `select_type` | Query type | SIMPLE | SUBQUERY, DERIVED |
| `table` | Table name | — | — |
| `type` | Access type | const/eq_ref/ref/range | ALL (full scan) |
| `possible_keys` | Possible indexes | Has value | NULL |
| `key` | Actually used index | Has value | NULL |
| `key_len` | Used index length | Reasonable range | Too large or too small |
| `ref` | Columns or constants used with index | Has value | NULL |
| `rows` | Estimated scanned rows | Small | Large |
| `filtered` | Percentage after WHERE filtering | 100 | Small |
| `Extra` | Extra info | Using index | Using filesort, Using temporary |

### type from Best to Worst

```
system > const > eq_ref > ref > fulltext > ref_or_null
  > index_merge > unique_subquery > index_subquery
  > range > index > ALL
```

Key red flags:
- **`ALL`**: Full table scan, usually needs optimization
- **`index`**: Full index scan (still scans all index rows, needs review)
- **`Using filesort`**: Requires additional sort operation
- **`Using temporary`**: Uses temporary table (usually GROUP BY or DISTINCT)

### EXPLAIN ANALYZE Example

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id;
```

Sample output:

```
-> Group aggregate: count(o.id)  (cost=5234 rows=1234)
   -> Left hash join (o.user_id = u.id)  (cost=4123 rows=12345)
      -> Filter: (u.created_at > TIMESTAMP '2025-01-01')  (cost=123 rows=500)
         -> Table scan on u  (cost=123 rows=1000)
      -> Hash
         -> Index scan on o using idx_user_id  (cost=2000 rows=10000)
```

The execution plan shows actual time spent at each step, helping you find bottlenecks.

---

## 3. Performance Schema

MySQL 5.7+ built-in performance monitoring tool.

### Enabling Performance Schema

```sql
-- Check if enabled
SHOW VARIABLES LIKE 'performance_schema';

-- Enable in my.cnf
-- performance_schema = ON
```

### Common Queries

```sql
-- 1. View most time-consuming SQL (by total execution time)
SELECT
    digest_text AS query,
    count_star AS exec_count,
    sum_timer_wait / 1000000000000 AS total_time_sec,
    avg_timer_wait / 1000000000000 AS avg_time_sec,
    sum_rows_examined / count_star AS avg_rows_examined
FROM performance_schema.events_statements_summary_by_digest
WHERE schema_name = 'shop'
ORDER BY sum_timer_wait DESC
LIMIT 10;

-- 2. View tables with highest IO pressure
SELECT
    object_schema,
    object_name,
    count_read,
    count_write,
    count_fetch,
    sum_timer_read / 1000000000000 AS read_time_sec
FROM performance_schema.table_io_waits_summary_by_table
ORDER BY sum_timer_read DESC
LIMIT 10;

-- 3. View lock wait situations
SELECT
    object_schema,
    object_name,
    index_name,
    count_star,
    count_read,
    count_insert,
    count_update,
    count_delete
FROM performance_schema.table_lock_waits_summary_by_table
ORDER BY count_star DESC
LIMIT 10;
```

---

## 4. SHOW PROFILE

Used to view detailed execution step times for SQL.

```sql
-- Enable profiling
SET profiling = 1;

-- Execute query
SELECT * FROM users WHERE email = 'test@test.com';

-- View time summary for all queries
SHOW PROFILES;
+----------+------------+--------------------------------------------------+
| Query_ID | Duration   | Query                                            |
+----------+------------+--------------------------------------------------+
|        1 | 0.00041200 | SELECT * FROM users WHERE email = 'test@test.com'|
+----------+------------+--------------------------------------------------+

-- View detailed step times for a specific query
SHOW PROFILE FOR QUERY 1;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000052 |
| checking permissions | 0.000005 |
| Opening tables       | 0.000012 |
| System lock          | 0.000006 |
| Waiting for query    | 0.000002 |
| init                 | 0.000018 |
| optimizing           | 0.000009 |
| statistics           | 0.000080 |  -- Statistics
| preparing            | 0.000012 |
| executing            | 0.000002 |
| Sending data         | 0.000106 |  -- Actual data sending
| end                  | 0.000003 |
| query end            | 0.000003 |
| closing tables       | 0.000007 |
| freeing items        | 0.000070 |
| cleaning up          | 0.000025 |
+----------------------+----------+
```

---

## 5. Monitoring Key Metrics

### System Status

```sql
-- View MySQL running status
SHOW GLOBAL STATUS;

-- Key metrics
-- Connections
SHOW GLOBAL STATUS LIKE 'Threads_%';

-- QPS (Queries Per Second)
SHOW GLOBAL STATUS LIKE 'Queries';

-- TPS (Transactions Per Second)
SHOW GLOBAL STATUS LIKE 'Com_commit';
SHOW GLOBAL STATUS LIKE 'Com_rollback';

-- Buffer pool hit rate
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
```

### Calculating Key Metrics

```sql
-- Calculate cache hit rate
SELECT
    (1 - (
        SELECT VARIABLE_VALUE FROM performance_schema.global_status
        WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
    ) / (
        SELECT VARIABLE_VALUE FROM performance_schema.global_status
        WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
    )) * 100 AS buffer_pool_hit_rate;

-- Calculate QPS
SELECT
    (VARIABLE_VALUE - @last_queries) / 10 AS qps
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Queries';
```

---

## 6. InnoDB Engine Monitoring

```sql
-- View InnoDB status (lots of key information)
SHOW ENGINE INNODB STATUS\G

-- Key focus areas:
-- 1. LATEST DETECTED DEADLOCK
-- 2. TRANSACTIONS (current active transactions)
-- 3. BUFFER POOL AND MEMORY (buffer pool usage)
-- 4. ROW OPERATIONS (row operation statistics)

-- View data table sizes
SELECT
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_mb,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb,
    table_rows
FROM information_schema.tables
WHERE table_schema = 'shop'
ORDER BY total_mb DESC;

-- View queries not using indexes
SELECT * FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NULL OR index_name = 'PRIMARY'
ORDER BY count_star DESC;
```

---

## 7. Common Performance Issues and Solutions

| Symptom | Possible Cause | Solution |
|------|---------|---------|
| CPU 100% | Full table scan, missing index | EXPLAIN analysis, add index |
| High IO | Buffer pool too small | Increase `innodb_buffer_pool_size` |
| Lock waits | Long transactions, high concurrency | Optimize transactions, reduce lock hold time |
| Temporary tables | GROUP BY/DISTINCT without index | Add index covering group fields |
| Large data volume | Deep pagination, not archived | Cursor pagination, periodic archiving |

### Common SQL Optimization Examples

```sql
-- 1. Original SQL (slow)
SELECT * FROM orders WHERE DATE(created_at) = '2025-01-15';

-- Optimization: avoid functions on indexed columns
SELECT * FROM orders
WHERE created_at >= '2025-01-15 00:00:00'
  AND created_at < '2025-01-16 00:00:00';


-- 2. Original SQL (slow)
SELECT * FROM users WHERE name LIKE '%张三%';

-- Optimization: cannot use index, consider full-text index
CREATE FULLTEXT INDEX idx_name ON users(name);
SELECT * FROM users WHERE MATCH(name) AGAINST('张三' IN BOOLEAN MODE);


-- 3. Original SQL (slow)
SELECT * FROM users ORDER BY RAND() LIMIT 10;

-- Optimization: avoid ORDER BY RAND()
SELECT * FROM users
WHERE id >= (SELECT FLOOR(MAX(id) * RAND()) FROM users)
ORDER BY id LIMIT 10;


-- 4. Original SQL (slow)
UPDATE orders SET status = 1 WHERE status = 0;
-- Updating a large amount of data at once

-- Optimization: batch update
UPDATE orders SET status = 1
WHERE status = 0 LIMIT 1000;
-- Loop in application
```

---

## 8. MySQL Configuration Optimization

```ini
[mysqld]
# Buffer pool -- most important config, set to 60-70% of memory
innodb_buffer_pool_size = 4G

# Log buffer
innodb_log_file_size = 512M
innodb_log_buffer_size = 64M

# IO threads
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# Dirty page flushing
innodb_io_capacity = 2000
innodb_flush_neighbors = 0          # SSD doesn't need neighbor flushing

# Connections
max_connections = 500
thread_cache_size = 100

# Temporary tables
tmp_table_size = 64M
max_heap_table_size = 64M

# Query cache (removed in MySQL 8.0, no need to configure)
# Sorting and grouping
sort_buffer_size = 2M
join_buffer_size = 2M
```

> Note: `sort_buffer_size` and `join_buffer_size` are session-level, don't set them too large (each connection gets its own allocation).

---

## 9. Optimization Process Summary

```
1. Enable slow query log → Find slow SQL
2. EXPLAIN analysis → Confirm the issue
3. Create or optimize indexes → Solve most problems
4. Optimize SQL writing → Solve remaining problems
5. Adjust MySQL configuration → Maximize hardware utilization
6. Data-level optimization → Partitioning, sharding, archiving
7. Architecture-level optimization → Read-write splitting, distribution
```

---

## Summary

Performance optimization starts with the slow query log, uses EXPLAIN as the analysis tool, and focuses on index optimization as the core approach. Most performance issues can be solved by adding appropriate indexes.

---

