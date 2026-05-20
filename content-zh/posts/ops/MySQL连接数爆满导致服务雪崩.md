---
title: "MySQL 连接数爆满导致服务雪崩——从 max_connections 到连接池优化"
date: 2026-05-21
weight: 100110
slug: "mysql-connections-full-crash"
tags: ["mysql", "database", "troubleshooting"]
categories: ["MySQL"]
description: "MySQL 连接数爆满导致全站不可用的完整排查，从误判 max_connections 到定位连接池泄漏和慢查询堆积"
keywords: "mysql too many connections, mysql max_connections, mysql连接池, 连接泄漏, wait_timeout"
draft: false
featured: true
cover:
  image: "/images/mysql-connections-banner.svg"
  caption: "MySQL 连接数爆满排查"
---

# MySQL 连接数爆满导致服务雪崩——从 max_connections 到连接池优化

## 问题现象

周三下午，告警突然炸了——核心业务接口全部报 500。查看应用日志：

```
2026-05-21 14:30:01.123 ERROR [http-nio-8080-exec-12] c.e.demo.controller.OrderController - Create order failed
com.zaxxer.hikari.pool.HikariPool$TimeoutException: HikariPool-1 - Connection is not available, request timed out after 30000ms
    at com.zaxxer.hikari.pool.HikariPool.createTimeoutException(HikariPool.java:696)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:201)
```

应用连接池拿不到连接，等了 30 秒超时后直接抛异常。

```bash
# 查 MySQL 当前连接数
mysql -u root -p -e "SHOW VARIABLES LIKE 'max_connections';"
```

```
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
```

```bash
# 当前连接数
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
```

```
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_connected | 151   |
```

151 连满了，新的请求进不来。

**影响**：所有依赖这个数据库的服务全部不可用——订单、支付、用户中心全挂了。

## 排查过程

### 错误尝试 1：增加 max_connections

看到连接满了，第一反应——"上限太低了，调大就行"。

```bash
# 临时调大连接数上限（不重启）
mysql -u root -p -e "SET GLOBAL max_connections = 500;"
```

连上来了，从 151 升到了 500。但过了 20 分钟，连接数又涨到了 480。继续涨。

```mysql
-- 观察连接数变化
SELECT COUNT(*) FROM information_schema.processlist;
```

```
+----------+
| COUNT(*) |
+----------+
|      480 |
```

按这个趋势，500 很快就满。

**踩坑点**：调大 `max_connections` 只是把天花板抬高了，没有解决"为什么需要这么多连接"的问题。而且每个连接都消耗内存（MySQL 每连接约 2-8 MB），调太大可能导致 OOM。

### 错误尝试 2：直接杀连接，治标不治本

```mysql
-- 查出所有非 Sleep 的连接
SELECT id, user, host, db, command, time, state, info
FROM information_schema.processlist
WHERE command != 'Sleep'
ORDER BY time DESC;
```

```
+------+------+-----------+--------+---------+------+-----------+------+
| id   | user | host      | db     | command | time | state     | info |
+------+------+-----------+--------+---------+------+-----------+------+
| 123  | app  | 10.0.0.1 | mydb   | Query   | 2345 | Sending data | SELECT * FROM orders WHERE status = 0 ORDER BY create_time DESC |
| 456  | app  | 10.0.0.2 | mydb   | Query   | 1890 | Sending data | SELECT * FROM orders WHERE status = 0 ORDER BY create_time DESC |
```

有些查询跑了 2000 多秒还没结束。手动 kill：

```mysql
KILL 123;
KILL 456;
```

连接数降了一点，但很快又上来了——应用会自动重试，新的慢查询又进来了。

然后我做了更蠢的事——大范围杀连接：

```mysql
-- 把所有非 Sleep 连接都杀了（除了自己）
SELECT CONCAT('KILL ', id, ';') FROM information_schema.processlist
WHERE user != 'root' AND command != 'Sleep';
```

这确实让连接数降了，但也把正常请求杀了，前端看到更多报错。而且应用做了重试逻辑，线程会自动重新连接，5 分钟后连接数又满了。

**踩坑点**：`KILL` 只是症状解。慢查询不解决、连接池不优化，杀了还会再来。而且无差别杀连接会影响正常业务。

### 错误尝试 3：重启 MySQL 服务

"那就重启 MySQL 吧。"这是我最后悔的操作。

```bash
systemctl restart mysqld
```

重启后连接清零了，服务恢复了。但好景不长——15 分钟后连接数又冲到了 200+。而且重启导致了两个后果：

1. **连接断开导致业务报错**：重启时所有连接被强制断开，正在执行的查询中断，应用报错
2. **缓存失效**：Buffer Pool 里的数据全丢了，重启后走了一波"缓存预热"，大量磁盘 IO，慢查询更多了

```bash
# 重启后的 IO 压力
iostat -x 1 3 | grep -E "Device|sda"
```

```
Device   r/s     w/s     rkB/s     wkB/s  await  %util
sda      2345.0  567.0   189000.0  45000.0  45.0  99.8
```

磁盘 util 99.8%，Buffer Pool 在疯狂从磁盘加载数据。

**踩坑点**：MySQL 重启是最后的手段，不能当常规恢复操作。重启后 Buffer Pool 冷启动会导致严重的性能下降，反而加剧问题。

### 真正的发现：定位到根源

```mysql
-- 查看当前活跃连接都在干什么
SELECT
    user,
    host,
    db,
    command,
    COUNT(*) as count,
    ROUND(AGgregate(TIME)) as avg_time_seconds
FROM information_schema.processlist
GROUP BY user, host, db, command
ORDER BY count DESC;
```

```
+------+-----------+--------+---------+------+-----------------+
| user | host      | db     | command | count| avg_time_seconds |
+------+-----------+--------+---------+------+-----------------+
| app  | 10.0.1.1  | mydb   | Query   | 45   | 120             |
| app  | 10.0.1.2  | mydb   | Query   | 38   | 95              |
| app  | 10.0.1.3  | mydb   | Query   | 42   | 110             |
| app  | 10.0.1.1  | mydb   | Sleep   | 12   | 300             |
```

40+ 个活跃查询，平均跑了一两分钟还没完。这不正常——正常的订单查询应该在 100ms 内返回。

```mysql
-- 查看慢查询日志
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';
```

```
+----------------+-----------------------------------+
| Variable_name  | Value                             |
+----------------+-----------------------------------+
| slow_query_log | ON                                |
| slow_query_log_file | /var/log/mysql/mysql-slow.log |
| long_query_time | 2.000000                         |
```

```bash
# 查最近 20 条慢查询
mysqldumpslow -s t -t 20 /var/log/mysql/mysql-slow.log
```

```
Count: 2345  Time=45.23s (106123s)  Lock=0.00s (23s)  Rows_sent=0.0 (0), Rows_examined=12345678.0 (28901234567)
  SELECT * FROM orders WHERE status = N ORDER BY create_time DESC

Count: 1234  Time=12.34s (15234s)  Lock=0.01s (12s)  Rows_sent=1000.0 (1234000), Rows_examined=5678901.0 (7000000000)
  SELECT * FROM order_items WHERE order_id IN (N, N, N, ...)
```

两条关键发现：

1. `SELECT * FROM orders WHERE status = 0 ORDER BY create_time DESC` —— 没有 `status` 和 `create_time` 的联合索引，全表扫描，平均 45 秒
2. `IN` 查询的 `order_id` 列表太长（几千个 ID），每次扫描几百万行

```mysql
-- 确认 orders 表的索引
SHOW INDEX FROM orders;
```

```
+--------+------------+----------+--------------+-------------+
| Table  | Non_unique | Key_name | Seq_in_index | Column_name |
+--------+------------+----------+--------------+-------------+
| orders |          0 | PRIMARY  |            1 | id          |
| orders |          1 | idx_user |            1 | user_id     |
+--------+------------+----------+--------------+-------------+
```

只有主键索引和 user_id 索引，没有 `(status, create_time)` 联合索引。这个查询每次扫全表，45 秒才能返回。同时大量这类查询并发执行，把连接池撑爆了。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | 慢查询堆积导致连接被长期占用，连接池耗尽 |
| 配置缺陷 | `max_connections` 只有 151，`wait_timeout` 默认 28800 秒（8 小时） |
| 根本原因 | orders 表缺少 `(status, create_time)` 联合索引，慢查询每次全表扫描 1200 万行|

慢查询 → 连接长期不释放 → 连接池满 → 新请求拿不到连接 → 业务报错 → 应用重试 → 更多连接进来 → 雪崩。

## 解决方案

### 方案 A：紧急止血——加索引（根治慢查询）

```mysql
-- 先加联合索引，慢查询立刻变快
ALTER TABLE orders ADD INDEX idx_status_create_time (status, create_time);
```

```bash
# 确认慢查询日志不再有 orders 全表扫描
mysqldumpslow -s t -t 5 /var/log/mysql/mysql-slow.log | grep -v "orders"
```

加索引后，原来 45 秒的查询降到了 30ms。

### 方案 B：临时降连接数（配合方案 A 使用）

```mysql
-- 调小 wait_timeout，让 Sleep 连接早点断开
SET GLOBAL wait_timeout = 300;
-- 5 分钟没有活动的连接自动断开

-- 调小 interactive_timeout（后台管理连接的超时）
SET GLOBAL interactive_timeout = 300;
```

但是注意：调小 `wait_timeout` 只能断开 Sleep 连接，不能解决活跃查询的问题。要结合方案 A 从源头减慢查询。

```mysql
-- 查看调整后的连接数变化
SELECT COUNT(*) FROM information_schema.processlist WHERE command = 'Sleep';
```

### 方案 C：配置连接池 max_connections（合理估算）

加完索引后，评估业务需要的真实连接数再设置：

```bash
# 应用连接池大小（HikariCP 配置）
# 每个微服务实例的 pool-size = 10
# 共 5 个实例，每个实例连同一个库
# 10 × 5 = 50 个连接就够
```

```mysql
-- 设置合理的上限（不是盲目调大）
SET GLOBAL max_connections = 200;
```

**三种方式对比**：

| 方案 | 适用场景 | 效果 | 风险 |
|------|----------|------|------|
| A. 加索引 | 慢查询导致连接耗尽 | ✅ 根治慢查询 | 加索引期间锁表（短暂）|
| B. 调 wait_timeout | Sleep 连接过多 | ✅ 释放空闲连接 | 应用长事务可能被断 |
| C. 调 max_connections | 连接数评估不合理 | ⚠️ 临时缓解 | 过高会导致 OOM |

### 验证恢复

```bash
# 1. 确认连接数下降
mysql -u root -p -e "SHOW STATUS LIKE 'Threads_connected';"
```
预期：回落到正常水平（50-100）。

```bash
# 2. 确认慢查询消失
tail -f /var/log/mysql/mysql-slow.log
```
预期：不再出现 orders 全表扫描。

```bash
# 3. 确认业务恢复正常
curl -I https://api.example.com/orders
```
预期：200 OK，响应时间 < 500ms。

✅ **恢复验证**：
- `Threads_connected` 稳定在 50-80
- 慢查询日志中 orders 的查询时间 < 100ms
- 业务监控不再报 connection timeout

## 长期预防

```bash
# 1. 慢查询监控告警
# 当单条查询超过 1 秒时告警
# MySQL: long_query_time = 1
# Prometheus: mysql_global_status_slow_queries > 10

# 2. 连接数监控
# Threads_connected / max_connections > 0.8 时告警

# 3. 定期索引评审
# 每季度 review 一次慢查询日志，找出缺少的索引

# 4. 应用连接池配置检查
# HikariCP 的 maximumPoolSize 要和 MySQL 的 max_connections 匹配
# 公式: max_connections >= 所有应用实例的 maximumPoolSize 之和 + 10（管理连接）
```

## 教训总结

1. **调大 max_connections 不是解决方案，只是把天花板抬高**。关键在于搞清楚"为什么需要这么多连接"。每个连接消耗 2-8 MB 内存，无脑调大可能导致 MySQL OOM。

2. **不要重启 MySQL 来释放连接**。重启会导致 Buffer Pool 冷启动，磁盘 IO 飙升，慢查询更多，恢复时间更长。重启是最后的手段，不是第一选择。

3. **杀连接只是症状解**。慢查询不解决、连接池不优化，杀了还会回来。无差别 kill 还会影响正常业务。

4. **慢查询是连接池问题的根因之一**。一条 45 秒的查询占着一个连接不放，十条并发就把连接池撑爆了。`EXPLAIN` 和 `SHOW INDEX` 是排查这类问题的核心工具。

5. **连接数估算公式**：MySQL `max_connections` ≥ 所有应用实例的 `maximumPoolSize` 总和 × 1.2（预留缓冲）。加索引前连接数 150+，加索引后 50 就够——差距就在那几十条慢查询。
