---
title: "MySQL 主从延迟引发生产故障 — 一个慢查询如何导致 30 分钟读写不一致"
date: 2026-06-07
weight: 100380
slug: "mysql-replication-lag-incident"
tags: ["mysql", "database", "troubleshooting", "replication", "performance"]
categories: ["Troubleshooting"]
description: "一次 MySQL 主从延迟生产故障纪实 — 一次 schema 变更导致从库慢查询，引发 30 分钟主从延迟，业务读写一致性被打破"
keywords: "MySQL 主从延迟, MySQL replication lag, Seconds_Behind_Master, 读写不一致, 从库延迟排查, MySQL 复制故障"
draft: false
featured: true
cover:
  image: ""
  caption: "MySQL 主从延迟生产故障排查"
---
# MySQL 主从延迟引发生产故障 — 一个慢查询如何导致 30 分钟读写不一致

## 常见搜索关键词 / Common Search Queries

| 中文 | English |
|------|---------|
| MySQL 主从延迟排查 | MySQL replication lag troubleshooting |
| Seconds_Behind_Master 持续增长 | Seconds_Behind_Master keeps increasing |
| 读写一致性 MySQL | Read-after-write consistency MySQL |
| MySQL 从库延迟原因 | MySQL slave lag cause |
| MySQL 并行复制配置 | MySQL parallel replication setup |
| pt-osc 主从延迟 | pt-online-schema-change replication lag |
| MySQL relay log 暴涨 | MySQL relay log space growing |
| 杀掉 MySQL 从库慢查询 | Kill long running query on MySQL replica |

---

## 故障经过 / The Incident

### 环境信息 / Environment

| 组件 | 规格 |
|------|------|
| MySQL 版本 | 8.0.32 (Community) |
| 复制模式 | 异步复制 (Async) |
| 复制格式 | 行格式 (RBR) |
| 拓扑结构 | 1 主 + 1 从 |
| 应用 A | 写入主库 |
| 应用 B | 读取从库 |
| 应用 C | 从库上运行分析查询 |
| 表大小 | ~1000 万行 |
| 存储引擎 | InnoDB |

### 时间线 / Timeline

| 时间 (UTC+8) | 事件 |
|--------------|------|
| 14:00 | 业务高峰期开始 |
| 14:15 | DBA 在主库执行 `ALTER TABLE orders ADD COLUMN` |
| 14:16 | 主库写入延迟飙升；DDL 通过 MDL 锁阻塞后续写入 |
| 14:18 | 从库 CPU 达到 100% |
| 14:20 | 客服首次收到用户反馈：订单提交成功但页面不显示 |
| 14:22 | 值班工程师被 paged |
| 14:25 | `SHOW REPLICA STATUS` 显示 `Seconds_Behind_Master` > 1800s |
| 14:28 | 根因定位：从库分析查询阻塞 SQL 线程 |
| 14:30 | 杀掉从库分析查询 |
| 14:32 | 从库开始追赶 |
| 14:45 | 主从延迟归零，服务完全恢复 |

### 症状 / Symptoms

- **用户反馈订单提交成功（通过 App A 写入主库）后刷新页面（App B 读取从库）看不到订单。**
- 从库服务器 CPU 打满到 100%。
- 从库磁盘 IO 等待飙升。
- 应用错误日志未记录数据库连接异常 — 查询正常返回，但数据是旧的。
- 从库的 `Relay_Log_Space` 暴涨到数 GB。

---

## 背景 / Background

### MySQL 异步复制原理 / MySQL Asynchronous Replication

MySQL 默认的异步复制模式下，主库将事务写入二进制日志（`binlog`），从库通过 I/O 线程拉取这些事件到中继日志（`relay-log`），然后由 SQL 线程顺序回放。

```
主库: [事务] → Binlog → 网络 → 
从库 I/O 线程: → Relay Log → 
从库 SQL 线程: → 存储引擎
```

**异步复制**的关键特性是：主库**不等待**从库确认事件已应用。在主库上提交的事务立即对主库读取者可见，但可能不会立即出现在从库上。这就是读写不一致的根本来源。

### `Seconds_Behind_Master` 的工作原理

`Seconds_Behind_Master` 是衡量复制延迟最常用的指标，但经常被误解：

- 它是 SQL 线程当前正在执行的事件的时间戳与从库当前系统时间之间的差值。
- 如果 SQL 线程在等待（没有事件可应用），它报告 `0`。
- 如果 SQL 线程正在积极应用事件且从库系统时钟快于主库时钟，该指标可能具有误导性。
- 它**不**衡量中继日志中待处理事件的数量。
- 在 MySQL 8.0 中，如果 SQL 线程未运行或配置了 `Replica_Delay`，可能显示 `NULL`。

### 二进制日志位置 vs. GTID

跟踪复制位置有两种方式：

| 方法 | 描述 | 优点 | 缺点 |
|------|------|------|------|
| **二进制日志位置** | `(binlog 文件名, 偏移量)` 对。每个 binlog 事件递增偏移量 | 简单，广泛理解 | 脆弱 — 如果主库 binlog 文件变化，坐标可能失效 |
| **GTID** (全局事务 ID) | `(源ID:事务ID)` 例如 `a1b2c3d4-1234-4321-aaaa-bbbbbbbb:1-500`。每个事务获得唯一 ID | 自愈 — 从库可自动跳过已应用的事务 | 设置略复杂 |

本次故障环境使用的是**基于位置的复制**。

---

## 排查过程 / Investigation

除非特别说明，以下命令均在从库执行。排查遵循逐步缩小范围的系统化方法。

### 第 1 步：检查复制状态 — `SHOW REPLICA STATUS\G`

怀疑主从延迟时，DBA 执行的第一个命令：

```sql
SHOW REPLICA STATUS\G
```

关键字段：

```
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
              Source_Log_File: mysql-bin.000147
          Read_Master_Log_Pos: 2891345678
               Relay_Log_File: relay-bin.000345
                Relay_Log_Pos: 451234567
        Relay_Source_Log_File: mysql-bin.000147
           Exec_Master_Log_Pos: 1098765432   ← 最后应用的事件位置
               Relay_Log_Space: 4781234567   ← ~4.7 GB 待处理中继日志
           Seconds_Behind_Master: 1832       ← 落后约 30 分钟！
```

即刻亮起的红灯：
- `Seconds_Behind_Master: 1832` — 从库落后超过 30 分钟。
- `Relay_Log_Space: 4781234567` — 近 4.7 GB 未应用中继日志堆积。
- `Read_Master_Log_Pos` (2,891,345,678) 与 `Exec_Master_Log_Pos` (1,098,765,432) 之差约为 **1.79 GB** 的 binlog 事件等待应用。这是原始积压量。

### 第 2 步：检查线程状态 — `SHOW PROCESSLIST;`

了解 SQL 线程为何落后：

```sql
SHOW PROCESSLIST;
```

| Id | User | Host | db | Command | Time | State | Info |
|----|------|------|----|---------|------|-------|------|
| 10 | system user | | | Connect | 0 | Waiting for source to send event | |
| 11 | system user | | | Connect | 1832 | Applying batch of row changes (update) | |
| 12 | app_user | 10.0.0.5:43210 | orders | Query | **812** | **Sending data** | `SELECT ... FROM orders WHERE ... GROUP BY ... ORDER BY ...` |
| 13 | app_user | 10.0.0.6:43211 | orders | Query | 2 | Creating sort index | `SELECT ... LIMIT 10` |

关键发现：**线程 12** 已运行 **812 秒**（约 13.5 分钟），处于 `Sending data` 状态 — 这是一个对 `orders` 表的重型分析查询。该查询消耗了从库几乎所有 CPU 和 IO 资源，导致 SQL 线程（线程 11）无法获得足够资源来应用 binlog 事件。

### 第 3 步：确认问题查询

```sql
SELECT THREAD_ID, PROCESSLIST_ID, THREAD_OS_ID, 
       PROCESSLIST_INFO, PROCESSLIST_TIME, PROCESSLIST_STATE
FROM performance_schema.threads
WHERE PROCESSLIST_ID = 12\G
```

查看完整查询：

```sql
SHOW FULL PROCESSLIST;
```

该分析查询为：

```sql
SELECT user_id, COUNT(*) as order_count, SUM(amount) as total_spent
FROM orders
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 90 DAY)
GROUP BY user_id
ORDER BY total_spent DESC
LIMIT 100;
```

该查询扫描了数百万行，对 `orders` 表进行了全表扫描或低效索引扫描，消耗了从库 CPU 超过 13 分钟。

### 第 4 步：检查 Binlog 格式

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

环境使用了**行格式复制 (RBR)**。这很重要因为：

- 使用 RBR 时，主库的 `ALTER TABLE` 生成基于行的 DDL 事件。
- 更重要的是，DDL 后的每个行变更事件都必须逐行复制。
- RBR 事件通常比基于语句的等价事件更大，导致积压时中继日志增长更快。

### 第 5 步：检查 SQL 线程状态

```sql
-- MySQL 8.0.22+:
SHOW REPLICA STATUS\G
```

查看 `Slave_SQL_Running_State`（在 MySQL 8.0.22 之后为 `Replica_SQL_Running_State`）：

```
Replica_SQL_Running_State: Applying batch of row changes (update)
```

确认 SQL 线程在积极工作（非等待状态），但遇到了性能瓶颈。

### 第 6 步：主库侧检查

在主库检查：

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

主库的 binlog 位置 (2,891,345,678) 与从库的 `Read_Master_Log_Pos` 一致，确认 I/O 线程正常工作。瓶颈完全在 SQL 线程侧。

### 第 7 步：估算延迟量级

```sql
-- 在主库执行
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

每个 binlog 文件 1 GB。未应用的事件横跨约 **1.79 GB** 的 binlog。在负载较高的从库上以典型回放速度（RBR 约 20-50 MB/s），这相当于 **30 分钟以上的积压**。

---

## 根因 / Root Cause

本次故障由一系列**叠加因素**共同导致：

### 1. `ALTER TABLE` 触发 — 元数据锁 (MDL) 争用

当 DBA 执行：

```sql
ALTER TABLE orders ADD COLUMN rebate_tier TINYINT DEFAULT 0;
```

MySQL 的 InnoDB 要求 DDL 操作持有**元数据锁 (MDL)**。`ALTER TABLE` 在 `orders` 表上获取了排他 MDL，阻塞了：

- 所有后续对 `orders` 的 `INSERT`、`UPDATE`、`DELETE` 语句（它们需要共享 MDL）。
- 所有后续 `SELECT ... FOR UPDATE` 语句。

在一个拥有约 1000 万行、繁忙的 OLTP 系统上，重建表（即使使用了 `ALGORITHM=INPLACE` 或 `INSTANT`，如果适用）会导致显著的写入排队。

### 2. 主库写入队列 → Binlog 爆炸

写入在 DDL 后面排队，主库的二进制日志积累了大量的批量行变更事件。一旦 DDL 完成且锁被释放，所有排队的写入在短时间内密集提交，产生了密集的 binlog 事件突发。

从库的 I/O 线程尽职地将所有这些事件拉入中继日志，但...

### 3. 重型分析查询饿死 SQL 线程

线程 12 上的分析查询（`SELECT ... GROUP BY ... ORDER BY ... LIMIT 100`）消耗了从库几乎所有可用 CPU（8 vCPU）。而 SQL 线程**默认是单线程的**，无法竞争到资源。从库 CPU 达到 100%，SQL 线程几乎无法推进。

### 4. 单线程 SQL 线程（关键瓶颈）

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

`replica_parallel_workers = 0` 表示**并行复制被禁用**。从库使用单个 SQL 线程回放 binlog 事件。

在 MySQL 8.0 中，并行复制（`replica_parallel_workers > 0`）允许从库并行应用来自不同数据库、甚至同一数据库的事务（使用 `LOGICAL_CLOCK`）。没有并行复制，从库上的一个分析查询就能拖垮整个复制管道。

### 5. `Seconds_Behind_Master` 的恶性循环

随着从库越来越落后：
- 中继日志增长（4.7 GB）。
- 中继日志读写的磁盘 IO 与分析查询的临时表写入竞争。
- 分析查询运行越久，积累的事件越多，追赶所需时间越长。

形成了一个**恶性循环**：

```
主库 DDL → MDL 阻塞写入 → 写入在 binlog 中爆发 →
从库 I/O 拉取所有事件 → 中继日志增长 →
分析查询消耗 CPU → SQL 线程被饿死 →
延迟增加 → 中继日志进一步增长 → 更多积压需要追赶
```

---

## 解决方案 / Resolution

### 紧急修复（立即执行）

**步骤 1：杀掉从库上的长时分析查询。**

```sql
-- 找到线程：
SHOW PROCESSLIST;

-- 杀掉查询（不中断连接）：
KILL QUERY 12;

-- 如果查询无法停止，杀掉连接：
KILL 12;
```

杀掉查询后，从库 CPU 从 100% 几乎立即降到约 30%。

**步骤 2：重置复制以加速追赶。**

```sql
STOP REPLICA;
START REPLICA;
```

这会重置 I/O 和 SQL 线程。有时重新启动可以清除内部缓冲区争用。在我们的案例中，杀掉查询后直接重启从库线程就足够了。

**步骤 3：监控延迟恢复。**

```sql
SHOW REPLICA STATUS\G
```

观察 `Seconds_Behind_Master` 逐步下降：

| 时间 | Seconds_Behind_Master |
|------|----------------------|
| 14:30 | 1832 |
| 14:33 | 1550 |
| 14:36 | 1120 |
| 14:39 | 680 |
| 14:42 | 210 |
| 14:45 | 0 |

**步骤 4：验证数据一致性。**

```sql
-- 在从库快速检查
SELECT COUNT(*) FROM orders WHERE rebate_tier = 0;

-- 比较主从校验和（抽样）
CHECKSUM TABLE orders;
```

从库在杀掉分析查询并重启从库线程后，大约花了 **15 分钟**完成追赶。

### 长期修复

**1. 启用并行复制**

```ini
# 从库 my.cnf
replica_parallel_workers = 4
replica_parallel_type = LOGICAL_CLOCK
replica_preserve_commit_order = 1
```

- `replica_parallel_workers = 4`：使用 4 个并行工作线程应用 binlog 事件。
- `replica_parallel_type = LOGICAL_CLOCK`：允许并行应用在主库同一逻辑时钟窗口内提交的事务（即不冲突的事务）。
- `replica_preserve_commit_order = 1`：确保事务在从库上按与主库相同的顺序提交，这对一致性至关重要。

启用并行复制后，单个分析查询对复制延迟的影响显著降低，即使一个工作线程暂时被饿死，其他工作线程也能继续推进。

**2. 使用 pt-online-schema-change 执行 Schema 变更**

[pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)（pt-osc）来自 Percona Toolkit，可以避免 DDL 阻塞写入：

```bash
pt-online-schema-change \
  h=localhost,D=production,t=orders \
  --alter "ADD COLUMN rebate_tier TINYINT DEFAULT 0" \
  --execute
```

pt-osc 的工作原理：
1. 创建带有新 Schema 的影子表。
2. 在原表上添加触发器以捕获变更。
3. 分块增量复制数据。
4. 通过 `RENAME TABLE` 原子性地交换表。

该方法仅在最终表交换的短暂瞬间持有 MDL 锁，将写入阻塞窗口从分钟级降至毫秒级。

**3. 将分析查询分离到专用只读从库**

本次触发故障的分析查询绝不应该运行在服务于实时用户流量的同一从库上。

推荐架构：

```
主库（写入）
 ├── 从库 A（服务 App B — 实时用户读取）
 └── 从库 B（服务 App C — 分析/BI 查询）
```

通过为分析工作负载专用一个独立的从库，重型 `GROUP BY` 和 `ORDER BY` 查询不会干扰复制或降低面向用户的读取性能。

**4. 实施主从延迟监控和告警**

```sql
-- 监控脚本（每 30 秒执行一次）：
SELECT
  NOW() AS check_time,
  VARIABLE_VALUE AS seconds_behind_master
FROM performance_schema.global_status
WHERE VARIABLE_NAME = 'Seconds_Behind_Master';
```

或使用 Prometheus + mysqld_exporter 配合告警规则：

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

建议告警阈值：
- **警告**：`Seconds_Behind_Master > 30` 持续 1 分钟
- **严重**：`Seconds_Behind_Master > 300` 持续 2 分钟

**5. 应用层读写一致性保障**

对于关键的写后读场景（如订单提交），应用应在写入后从主库读取：

```python
# 之前（可能从从库读到脏数据）：
def submit_order(user_id, items):
    order_id = db_master.execute("INSERT INTO orders ...")
    order = db_replica.query("SELECT * FROM orders WHERE id = ?", order_id)  # BUG!
    return order

# 之后（从主库读取确保一致性）：
def submit_order(user_id, items):
    order_id = db_master.execute("INSERT INTO orders ...")
    order = db_master.query("SELECT * FROM orders WHERE id = ?", order_id)  # OK
    return order

# 或：带延迟容忍的写后读：
def submit_order(user_id, items):
    order_id = db_master.execute("INSERT INTO orders ...")
    time.sleep(0.5)  # 短暂等待复制
    order = db_replica.query("SELECT * FROM orders WHERE id = ?", order_id)
    return order
```

更优雅的方式是使用**会话一致性**或**读你所写一致性**模式：

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

## 经验教训 / Lessons Learned

### 技术教训

| # | 教训 | 采取的行动 |
|---|------|-----------|
| 1 | 大表上的 DDL 操作通过 MDL 阻塞写入 | 所有 Schema 变更采用 pt-online-schema-change |
| 2 | MySQL 默认复制是单线程的，非常脆弱 | 在所有从库上启用并行复制 |
| 3 | 从库上的重型分析查询会拖慢复制速度 | 在从库级别分离 OLTP 和 OLAP 流量 |
| 4 | 仅靠 `Seconds_Behind_Master` 不足以全面监控 | 增加 `Relay_Log_Space`、`Exec_Master_Log_Pos` 差距和复制吞吐量指标 |
| 5 | RBR 生成比 SBR 更大的 binlog 事件，影响负载下的追赶速度 | 规划从库容量时考虑 binlog 格式 |
| 6 | 没有监控，主从延迟可能长时间不被发现 | 部署分级严重程度的延迟告警 |

### 流程教训

- **变更管理**：`ALTER TABLE` 在业务高峰期执行，未经事先审查。所有 Schema 变更应遵循变更管理流程，规划执行窗口（最好在非高峰期）。
- **应急预案**：团队此前没有文档化的主从延迟故障应急预案。事后创建了涵盖检测、排查和紧急响应步骤的预案文档。
- **测试**：重型分析查询未曾在真实数据量下测试。预发布环境应镜像生产数据量以进行查询性能测试。

---

## 总结 / Summary

### 故障时间线总结 / Incident Timeline Summary

```
时间        事件
─────       ───────────────────────────────────────────────────────────
14:15       DBA 在主库执行 ALTER TABLE orders ADD COLUMN（1000 万行）
14:16       主库 MDL 锁阻塞写入；写入队列堆积
14:18       从库 CPU 被分析查询打满 100%（该查询自 14:14 开始运行）
14:20       用户反馈刷新后订单不显示
14:22       值班工程师被 paged
14:25       SHOW REPLICA STATUS → Seconds_Behind_Master = 1832s
14:28       定位根因：分析查询饿死 SQL 线程
14:30       在从库执行 KILL QUERY 12；CPU 下降；重启从库线程
14:33       延迟开始下降（1550s → 1120s → 680s ...）
14:45       延迟归零；服务完全恢复
```

### 峰值关键指标 / Key Metrics at Peak

| 指标 | 数值 |
|------|------|
| `Seconds_Behind_Master` | 1832s（约 30 分钟） |
| `Relay_Log_Space` | ~4.7 GB |
| Binlog 积压量 | ~1.79 GB |
| 分析查询运行时间 | 被杀前已运行 812s |
| 从库 CPU | 100% |
| `replica_parallel_workers` | 0 |
| 恢复时间（杀查询 → 追赶完成） | ~15 分钟 |

### 预防措施总结 / Preventive Measures Summary

| 措施 | 状态 | 负责人 |
|------|------|--------|
| 启用并行复制 (`replica_parallel_workers=4`) | 已完成 | DBA |
| 所有 DDL 采用 pt-online-schema-change | 已完成 | DBA 团队 |
| 为分析查询新增专用从库 | 进行中 | 基础架构 |
| 部署主从延迟监控 + 告警 | 已完成 | 可观测性 |
| 在订单服务中实现读你所写模式 | 进行中 | 后端团队 |
| 创建主从延迟故障应急预案 | 已完成 | SRE |
| 将 ALTER TABLE 审查加入变更管理流程 | 已完成 | 工程经理 |

本次故障最重要的启示是：**主从延迟不仅仅是数据库指标 — 它是直接影响用户的问题**。30 分钟的延迟意味着 30 分钟内用户看到自己的订单"消失"，这侵蚀信任并直接影响业务指标。每个运行主从复制拓扑 MySQL 的团队都应该像对待可用性监控一样对待主从延迟监控。

---

*写于 2026 年。MySQL 8.0.32。本文属于 [blog.777157.xyz](https://blog.777157.xyz) 生产故障排查系列。*
