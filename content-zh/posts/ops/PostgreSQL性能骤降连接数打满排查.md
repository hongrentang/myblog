---
title: "PostgreSQL性能骤降连接数打满——一次索引缺失导致的数据库雪崩排查实录"
date: 2026-06-06
weight: 100360
slug: "postgresql-performance-crash-connection-pool"
tags: ["postgresql", "database", "troubleshooting", "performance", "connection"]
categories: ["Troubleshooting"]
description: "PostgreSQL 生产环境性能骤降排查实录——因缺失联合索引导致全表扫描、连接池打满、数据库雪崩的完整排查过程，涉及 pg_stat_activity 和 pg_stat_statements 实战分析"
keywords: "PostgreSQL 性能排查, PostgreSQL 连接打满, pg_stat_activity, pg_stat_statements, 索引缺失, 全表扫描, PostgreSQL 故障处理"
draft: false
featured: true
cover:
  image: ""
  caption: "PostgreSQL 性能骤降——连接数打满排查"
---

## 常见搜索关键词

- PostgreSQL 连接池耗尽
- PostgreSQL max_connections 打满
- PostgreSQL 性能突然变慢
- PostgreSQL pg_stat_activity 全部活跃
- PostgreSQL 慢查询排查
- PostgreSQL 缺失索引导致全表扫描
- pg_stat_statements 性能分析
- PostgreSQL 连接风暴 雪崩效应
- PostgreSQL 杀死长时间运行的查询
- PostgreSQL CREATE INDEX CONCURRENTLY 不锁表

---

## 故障经过

**环境信息：**

| 组件 | 规格 |
|------|------|
| PostgreSQL 版本 | 15.4 |
| 内存 | 16 GB |
| vCPU | 4 核 |
| 存储 | 500 GB SSD |
| max_connections | 200 |
| 操作系统 | Ubuntu 22.04 LTS |

这是一个普通的周二上午。10:00 整——每日业务高峰——监控系统开始疯狂告警。

**故障时间线：**

| 时间 | 事件 |
|------|------|
| 10:01 | Prometheus 告警：PostgreSQL 连接数达到 95%（190/200） |
| 10:02 | API 响应时间从 50ms 飙升到 15s |
| 10:03 | 应用服务器开始返回 HTTP 500 错误 |
| 10:04 | 数据库 CPU 使用率 100%，iowait 高达 60% |
| 10:05 | 出现 `FATAL: remaining connection slots are reserved for non-replication superuser connections` |
| 10:06 | 值班告警触发——应急响应启动 |

到 10:05，生产数据库实际上已经宕机。集群中所有应用服务器都无法获取数据库连接。少数侥幸获取到连接的查询也需要 20-30 秒才能完成——相比正常时的 50ms，性能下降了 400-600 倍。

---

## 背景

故障发生前两天，产品团队上线了一个新功能：**订单仪表盘 v2**。该功能引入了一个新的 API 端点 `/api/v2/orders/summary`，允许运营人员按用户 ID、订单状态和日期范围查询订单汇总。

查询语句如下：

```sql
SELECT user_id, status, count(*) as order_count, sum(amount) as total_amount
FROM orders
WHERE user_id = 12345
  AND status IN ('pending', 'processing', 'shipped')
  AND created_at >= '2026-06-01'
GROUP BY user_id, status
ORDER BY status;
```

ORM（SQLAlchemy）从高层模型查询生成了这条 SQL。代码评审时，团队重点审查了业务逻辑的正确性——状态流转、金额计算、分页等。没有人检查查询计划。

`orders` 表有 **500 万行数据**。表上只有单独的 `user_id` 索引和 `created_at` 索引，**没有包含全部三个过滤字段的联合索引**。PostgreSQL 的查询规划器找不到高效的执行路径，于是选择了对 500 万行数据进行**顺序扫描**。

在预发环境中这个问题没有被发现，原因是：
- 预发环境只有 5 万行数据（不是 500 万）
- 预发环境没有并发负载
- 预发环境查询只需 50ms，掩盖了问题

---

## 排查过程

### 第一步：检查连接数

问题的第一个信号很明显——应用无法连接。我们直接登录到数据库：

```sql
SELECT count(*) FROM pg_stat_activity;
```

```
 count
-------
   200
(1 row)
```

全部 200 个连接槽位已被占满。查看各连接的状态：

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

180 个连接处于 `active` 状态——意味着 180 个查询正在活跃运行（或阻塞）。正常情况下，连接池中大约只有 20 个活跃连接，其余应为空闲状态。

### 第二步：查找长时间运行的查询

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

模式一目了然——同一个查询在数十个连接上同时运行，每个耗时 5-8 分钟。等待事件 `DataFileRead` 表明大量 IO 操作正在将数据块从磁盘读取到共享缓冲区——这是大表顺序扫描的典型特征。

### 第三步：使用 pg_stat_statements 分析查询性能

我们检查了 `pg_stat_statements` 视图来了解查询的累计性能：

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

**排查结果触目惊心：**

- 问题查询累计执行时间达 **28,341,920 ms**（近 8 小时的累积 CPU 时间）
- 平均执行时间：**1,682 ms**（每次查询 1.7 秒）
- **命中率：0.01**——仅 1% 的数据块在共享缓冲区中找到，意味着 99% 的读取来自磁盘
- 相比之下，其他所有查询的命中率都在 0.998 以上，平均执行时间低于 200ms
- 自新版本上线以来，该查询已被调用 **16,854 次**

### 第四步：EXPLAIN ANALYZE 慢查询

我们获取了确切的查询文本并运行 `EXPLAIN (ANALYZE, BUFFERS)`：

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

**从执行计划中发现的关键问题：**

1. **Seq Scan on orders**——PostgreSQL 扫描了 orders 表的全部 **500 万行**数据
2. **Rows Removed by Filter: 5,000,000**——每次查询读取并丢弃了 500 万行数据
3. **Execution Time: 5,850 ms**——每次查询近 6 秒
4. **Buffers: shared hit=384 read=49512**——从磁盘读取了 49,512 个数据块（约 387 MB/次）
5. 查询只返回了 **3 行**数据——大海捞针

### 第五步：检查现有索引

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

表上有单独的 `user_id` 索引和 `created_at` 索引。然而，PostgreSQL 的查询规划器在这种场景下无法高效地组合两个独立的 B-tree 索引——它需要同时过滤 `user_id`、`status` 和 `created_at` 三个字段。没有包含这三列的**联合索引**，规划器退而求其次选择了顺序扫描，因为优化器判断组合两个独立索引的开销不值得。

**为什么单独索引不够用：** PostgreSQL 可以通过位图扫描组合两个索引（bitmap-and），但由于多了 `status` 过滤条件以及 `GROUP BY`/`ORDER BY` 的要求，优化器得出结论：全表扫描比位图组合方式成本更低。对于一个 500 万行的表来说，这个决策是灾难性的。

---

## 根因

1. **缺少 (user_id, status, created_at) 联合索引**——新查询需要过滤三个字段，但只有各自的单列索引，导致每次查询都需要全表扫描 500 万行数据。

2. **每次执行都进行全表扫描**——每次查询从磁盘读取约 387 MB 数据，耗时 5-8 秒。50+ 并发请求使 IO 子系统彻底饱和。

3. **连接不断累积**——随着查询变慢（每次 5-8 秒），连接被保持的时间越来越长。180 个活跃连接全部等待 IO，连接池被耗尽。

4. **max_connections=200 的天花板**——一旦 200 个连接槽位全部被占满，新连接被拒绝并返回致命错误。应用服务器收到错误后立即重试，形成了**惊群效应**，导致不人工介入就无法恢复。

5. **CI/CD 中没有查询审查环节**——部署流水线没有步骤来分析查询计划或检测大表上的顺序扫描。预发环境的小数据集完全掩盖了问题。

---

## 解决方案

### 紧急处理（10:06 - 10:12）

**第一步：杀掉长时间运行的查询，释放连接。**

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

这条命令立即杀掉了 178 个查询。连接数从 200 降到了 22。

**第二步：临时增加 max_connections 应对突发流量。**

```sql
ALTER SYSTEM SET max_connections = 300;
```

然后重启 PostgreSQL 服务：

```bash
sudo systemctl restart postgresql
```

> **注意：** 修改 `max_connections` 需要完全重启数据库，会导致短暂停机。在我们的场景中，数据库已经不可用，因此 30 秒的重启是可以接受的。未来考虑使用 PgBouncer 来复用连接以避免此类问题。

**第三步：确认数据库恢复正常。**

```sql
SELECT count(*) FROM pg_stat_activity;
```

```
 count
-------
    18
(1 row)
```

API 响应时间恢复正常（50ms）。

### 永久修复（10:30 - 10:45）

**创建缺失的联合索引。**

使用 `CREATE INDEX CONCURRENTLY` 避免锁表：

```sql
CREATE INDEX CONCURRENTLY idx_orders_user_status_created
ON orders (user_id, status, created_at);
```

在 500 万行数据的表上，这个操作耗时约 4 分钟（不阻塞读写）。

**验证创建索引后的查询计划：**

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

**修复前后对比：**

| 指标 | 修复前（顺序扫描） | 修复后（索引扫描） | 提升幅度 |
|------|-------------------|--------------------|----------|
| 执行时间 | 5,850 ms | 0.082 ms | **71,000 倍** |
| 读取缓冲区 | 49,512 个磁盘块 | 6 个共享缓冲区 | **8,252 倍 IO 降低** |
| 扫描行数 | 5,000,000 | 8 | **625,000 倍** |
| 命中率 | 0.01（1%） | 1.0（100%） | **完美缓存命中** |
| 查询类型 | 顺序扫描 | 仅索引扫描 | — |

### 应用层改进

数据库稳定后，我们在应用层做了几项改进：

1. **减小连接池大小**——每个应用实例配置了 20 个连接。10 个实例就是 200 个连接——正好是上限。我们配合 PgBouncer 的事务模式将连接数降至每个实例 5 个：

   ```python
   # SQLAlchemy 配置 - 修复前
   engine = create_engine(DB_URL, pool_size=20, max_overflow=10)
   
   # SQLAlchemy 配置 - 修复后（配合 PgBouncer）
   engine = create_engine(DB_URL, pool_size=5, max_overflow=2)
   ```

2. **添加指数退避重试**——应用代码更新为在连接失败时带抖动重试：

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
       raise Exception("超出最大重试次数")
   ```

3. **添加查询超时**——作为防止未来失控查询的安全网：

   ```sql
   ALTER DATABASE mydb SET statement_timeout = '30s';
   ```

4. **CI/CD 查询审查**——我们在 CI 流水线中增加了一个自动化步骤，对所有新查询在生产级数据集上运行 `EXPLAIN`，如果检测到对超过 10 万行的大表进行顺序扫描，则构建失败。`pg_qualstats` 和 `auto_explain` 等工具也可以帮助在预发环境检测问题查询。

---

## 经验教训

### 出了什么问题

1. **新查询没有经过数据库审查**——代码评审流程检查了业务逻辑但没有检查查询性能。一个简单的 `EXPLAIN` 就能发现问题。
2. **预发环境数据量不匹配**——预发环境只有 5 万行，而生产环境是 500 万行。查询在预发环境表现良好，给了错误的信心。
3. **没有查询性能基线**——我们有系统指标（CPU、内存、连接数）的监控，但没有查询级别的性能监控。`pg_stat_statements` 虽然启用了，但没有人主动查看。
4. **没有连接池中间件**——应用程序直接连接 PostgreSQL，没有使用 PgBouncer 或类似的中间件，使连接耗尽成为单点故障。
5. **没有语句超时设置**——没有 `statement_timeout`，失控的查询可以无限运行，一直占用连接。

### 我们改进了什么

| 领域 | 改进前 | 改进后 |
|------|--------|--------|
| 查询审查 | 仅人工代码评审 | CI/CD 自动 EXPLAIN 检查 |
| 连接管理 | 每个应用独立连接池 | PgBouncer 事务级连接池 |
| 监控 | 仅系统指标 | pg_stat_statements 看板 + 查询性能告警 |
| 预发环境数据 | 5 万行 | 脱敏的生产数据快照（500 万行） |
| 查询安全 | 无超时 | statement_timeout = 30s |
| 故障响应 | 无预案 | 创建数据库故障应急手册 |
| 容量规划 | 无连接预估 | 按服务记录连接预算 |

### 检测 vs 预防

最有价值的教训：**我们检测到了症状（连接打满）但没有检测到原因（慢查询）**。如果我们对查询执行时间和顺序扫描有监控，我们会在部署后几分钟内发现这个问题——而不是两天后生产环境宕机时才意识到。

---

## 总结

这次事故是**缺失索引**如何级联导致全站瘫痪的一个教科书案例：

```
新上线查询缺少联合索引
         │
         ▼
  500 万行全表顺序扫描
         │
         ▼
  每次查询耗时 5-8 秒
         │
         ▼
  50 并发 → 连接被长期占用
         │
         ▼
  max_connections=200 耗尽
         │
         ▼
  新连接被拒绝 →
  应用重试 → 惊群效应
         │
         ▼
    数据库瘫痪 ❌
```

修复方案很简单——创建一个联合索引——但影响是全站停机 45 分钟。核心要点：

- **新查询上线前务必 `EXPLAIN`**，尤其是涉及大表的查询
- **使用 `pg_stat_statements`** 作为查询性能回退的早期预警系统
- **部署 PgBouncer** 将应用连接与数据库连接解耦
- **设置 `statement_timeout`** 作为安全网
- **预发环境数据量要与生产环境匹配**（或使用真实数据集）
- **监控查询性能**，而不仅仅是系统指标

联合索引将查询时间从 5,850 ms 降至 0.082 ms——**71,000 倍的性能提升**。PostgreSQL 是强大的数据库，但仅靠优化本身无法弥补缺失索引的问题。查询规划器的效果取决于你给它提供了什么样的索引。

**最后的话：** 每次你写 `WHERE` 子句时，想想你的索引。每次你评审代码时，不要只检查逻辑——检查查询计划。一个索引的缺失，可能就是生产环境的灾难。

---

## 参考资料

- [PostgreSQL 文档：使用 EXPLAIN](https://www.postgresql.org/docs/15/using-explain.html)
- [PostgreSQL 文档：pg_stat_statements](https://www.postgresql.org/docs/15/pgstatstatements.html)
- [PostgreSQL 文档：CREATE INDEX CONCURRENTLY](https://www.postgresql.org/docs/15/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)
- [PostgreSQL 文档：连接设置](https://www.postgresql.org/docs/15/runtime-config-connection.html)
- [PostgreSQL Wiki：索引维护](https://wiki.postgresql.org/wiki/Index_Maintenance)
- [PgBouncer 文档](https://www.pgbouncer.org/)
