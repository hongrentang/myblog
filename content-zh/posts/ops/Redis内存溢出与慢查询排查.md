---
title: "Redis 内存溢出与慢查询排查——无界缓存如何击垮会话存储"
date: 2026-06-08
weight: 100400
slug: "redis-memory-overflow-slowlog"
tags: ["redis", "database", "troubleshooting", "memory", "performance"]
categories: ["Troubleshooting"]
description: "一次 Redis 内存溢出事件——应用程序缺陷导致键无限增长，触发 maxmemory-policy 逐出循环、慢查询告警，最终导致会话存储完全失效"
keywords: "redis 内存溢出, redis maxmemory, redis 逐出策略, redis 慢查询, redis oom, redis keyspace 分析"
draft: false
featured: true
cover:
  image: ""
  caption: "Redis 内存溢出排查实录"
---

## 常见搜索关键词

- Redis OOM command not allowed when used memory > maxmemory
- Redis evicted_keys 突然飙升
- Redis slowlog 显示大量慢命令
- Redis used_memory 接近 maxmemory
- redis-cli --bigkeys 扫描分析
- Redis 会话存储逐出循环
- Redis maxmemory-policy 故障排查
- Redis MEMORY DOCTOR 报告解读
- Redis keyspace 分析与清理
- Redis CPU 100% slowlog 排查

---

## 故障经过

### 环境信息

| 组件 | 详情 |
|------|------|
| **Redis 版本** | 6.2.6（单机模式，无复制） |
| **最大内存** | 8 GB |
| **逐出策略** | `allkeys-lru` |
| **工作负载** | Node.js Web 应用的会话存储 |
| **流量** | 峰值约 20,000 req/s |
| **操作系统** | Ubuntu 20.04, 16 vCPU, 32 GB RAM |
| **部署方式** | EC2 (c5.4xlarge) 自运维 |

### 时间线

**14:30 — 代码部署**

一次例行的后端部署包含了会话中间件的改动。一次善意的重构引入了一个在每次请求上追加随机 UUID 的 `sessionId` 生成器。应用程序没有为用户的会话复用现有会话 ID，而是为几乎每个 HTTP 请求创建了一个全新的键——`session:a1b2c3d4-...`。

简化后的代码大致如下：

```javascript
// 有问题的代码：每次请求生成新的会话键
app.use(session({
  genid: () => uuid.v4(),    // <-- 每次请求生成新的 UUID！
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret'
}));
```

**15:10 — 首次告警**

PagerDuty 触发告警。应用程序延迟 P99 从 50 ms 飙升到 2,800 ms。Redis 命令延迟从约 2 ms 飙升到 600+ ms。

**15:15 — 用户影响**

用户开始报告"会话已过期"错误和意外的登出。工单队列爆炸。

**15:20 — 排查开始**

SRE 团队连接到 Redis 实例后发现：

```
OOM command not allowed when used memory > maxmemory
```

Redis 已经耗尽了 8 GB 的 `maxmemory`，开始拒绝写入。

---

## 背景

### Redis 内存模型

Redis 将所有数据存储在内存中。当配置了 `maxmemory` 后，Redis 会跟踪总内存使用量，并在达到限制时执行逐出策略。关键内存区域包括：

- **used_memory**: Redis 分配的总字节数（包括开销）
- **used_memory_rss**: 操作系统看到的字节数（常驻集大小）
- **used_memory_peak**: 历史峰值使用量
- **used_memory_lua**: Lua 脚本内存
- **used_memory_overhead**: 键值元数据使用的内存（非数据本身）
- **used_memory_dataset**: 实际数据内存
- **used_memory_dataset_perc**: 数据集内存占净使用量的百分比
- **maxmemory**: 配置的硬限制

### 逐出策略

Redis 提供多种 `maxmemory-policy` 选项：

| 策略 | 行为 |
|------|------|
| `noeviction` | 达到内存限制时，写入操作返回错误 |
| `allkeys-lru` | 从所有键中逐出最近最少使用的键 |
| `allkeys-lfu` | 从所有键中逐出最不常使用的键 |
| `volatile-lru` | 从设置了 TTL 的键中逐出 LRU 键 |
| `volatile-lfu` | 从设置了 TTL 的键中逐出 LFU 键 |
| `allkeys-random` | 随机逐出键 |
| `volatile-random` | 从设置了 TTL 的键中随机逐出 |
| `volatile-ttl` | 逐出 TTL 最短的键 |

默认的 `noeviction` 是安全的但可能导致写入失败。`allkeys-lru` 虽然常用于缓存场景，但对于会话存储来说可能很危险——它无法区分活跃会话和垃圾键。

### Redis 慢查询日志

`SLOWLOG` 记录执行时间超过配置阈值（`slowlog-log-slower-than`，默认 10,000 微秒 = 10 ms）的命令。日志存储在内存中（可通过 `slowlog-max-len` 配置，默认 128 条）。

```
SLOWLOG GET 20
```

返回条目包含：

- **id**: 唯一条目 ID
- **timestamp**: Unix 时间戳
- **microseconds**: 执行时长
- **command**: 命令及其参数
- **client IP:port**: 来源客户端
- **client name**: 可选的客户端名称

---

## 排查过程

### 第 1 步：检查 Redis 内存使用

首先登录 Redis 实例并运行 `INFO memory`：

```
$ redis-cli INFO memory

# Memory
used_memory:8589934592
used_memory_human:8.00G
used_memory_rss:8620007424
used_memory_peak:8590000128
used_memory_peak_human:8.00G
used_memory_lua:37888
maxmemory:8589934592
maxmemory_human:8.00G
maxmemory_policy:allkeys-lru
mem_fragmentation_ratio:1.00
mem_allocator:jemalloc-5.1.0
```

**发现**: `used_memory` 等于 `maxmemory`——实例已经完全占满。碎片率 1.00 表示没有显著的内存碎片（此时几乎所有分配都被使用中）。

### 第 2 步：检查逐出统计

```
$ redis-cli INFO stats | grep evicted

evicted_keys:2147593
evicted_keys_per_second:298.33
```

**发现**: 超过 210 万个键已被逐出。峰值时 Redis 每秒逐出约 300 个键。这解释了 CPU 高的原因——每次逐出都涉及 LRU 采样、键删除和内存释放，这是 CPU 密集型操作，尤其是在持续的写入压力下。

### 第 3 步：检查慢查询日志

```
$ redis-cli SLOWLOG GET 20

1) 1) (integer) 4021
   2) (integer) 1717780800
   3) (integer) 342791
   4) 1) "SETEX"
      2) "session:anon:a1b2c3d4-e5f6-7890-abcd-ef1234567890"
      3) "3600"
      4) "{\"cookie\":...}"
   5) "192.168.1.50:43802"
   6) ""

2) 1) (integer) 4020
   2) (integer) 1717780799
   3) (integer) 289456
   4) 1) "GET"
      2) "session:auth:user-98765"
   5) "192.168.1.51:43901"
   6) ""
```

**发现**：
- `session:anon:*` 键的 `SETEX` 命令耗时 342 ms（远超默认的 10 ms 阈值）
- 合法会话键的 `GET` 命令耗时 289 ms——正常情况下这些 <1 ms
- 慢查询日志被匿名会话键的 `SETEX` 和认证会话键的 `GET` 主导

`SETEX` 慢是可以预期的（逐出 + 分配颠簸导致），但 `GET` 慢令人担忧——这表明整个 Redis 性能都在下降，不仅仅是写入密集型操作。

### 第 4 步：分析键空间

```
$ redis-cli INFO keyspace

# Keyspace
db0:keys=2187642,expires=128,avg_ttl=0
```

**发现**: 数据库 0 中有 218 万个键，但只有 128 个键设置了 TTL。`avg_ttl` 为 0 证实了绝大多数键是**永久的**——它们永远不会自动过期。

### 第 5 步：查找键模式

```
$ redis-cli SCAN 0 MATCH session:* COUNT 10000

1) "235672"
2) 1) "session:anon:a1b2c3d4-e5f6-7890-abcd-ef1234567890"
   2) "session:anon:b2c3d4e5-f6a7-8901-bcde-f12345678901"
   3) "session:auth:user-54321"
   ...
```

即使设置 `COUNT 10000`，`SCAN` 也需要数秒才能完成。键的分布很清晰：

- **`session:anon:*`**: 约 210 万个键——这些是每次请求创建的有问题的键
- **`session:auth:*`**: 约 8 万个键——合法的认证会话键

垃圾键与合法键的比例约为 26:1。

### 第 6 步：MEMORY STATS 和 MEMORY DOCTOR

```
$ redis-cli MEMORY STATS

1) "peak.allocated"
2) (integer) 8590000128
3) "total.allocated"
4) (integer) 8589934592
5) "startup.flush"
6) (integer) 12496848
7) "overhead.total"
8) (integer) 3847269376
9) "keys.count"
10) (integer) 2187642
11) "keys.bytes-per-key"
12) (integer) 178
13) "dataset.bytes"
14) (integer) 4742665216
15) "dataset.percentage"
16) (percentage) 55.22
17) "peak.percentage"
18) (percentage) 99.99
...
```

```
$ redis-cli MEMORY DOCTOR

Memory doctor report:
- High overhead: 38% of used_memory is overhead, which is high.
- Keys with TTL: 128 out of 2187642 keys have TTL set, which is less than 0.01%.
- Peak memory: This instance used 99.99% of its peak memory.
- **OOM detected**: The server is configured with a maxmemory limit and the used memory is already at the limit.
- Recommendation: 1. Set TTLs on keys to allow automatic eviction.
  2. Reduce memory overhead by using smaller keys or data structures.
  3. Add more memory or switch to a Redis Cluster.
```

`MEMORY DOCTOR` 提供了简洁、人类可读的诊断结果，证实了我们的猜测。

### 第 7 步：检查客户端连接

```
$ redis-cli CLIENT LIST

id=43921 addr=192.168.1.50:43802 fd=31 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=2056 events=r cmd=SETEX user=default
id=43922 addr=192.168.1.51:43901 fd=32 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=2056 events=r cmd=GET user=default
id=43923 addr=192.168.1.52:44012 fd=33 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=2056 events=r cmd=SETEX user=default
...
```

**发现**: 数十个连接堆积着 `cmd=SETEX` 和 `cmd=GET`。部分连接显示 `omem`（输出内存）值在增长——Redis 难以跟上写入负载，导致输出缓冲区不断积累。

### 第 8 步：redis-cli --bigkeys 扫描

```
$ redis-cli --bigkeys

# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

-------- summary -------

Sampled 2187642 keys in the keyspace!
Total key length in bytes is 41095878 (avg len 18.78)

Biggest string found "session:anon:..." has 842 bytes
Biggest list  found has 1 items
Biggest set   found has 0 members
Biggest hash  found has 0 fields
Biggest zset  found has 0 members

2187641 strings with 4948264192 bytes (99.99% of keys, avg size 2.26 KB)
00 lists with 00 items
00 sets with 00 members
00 hashs with 00 fields
00 zsets with 00 members
```

**发现**: 几乎所有键都是字符串（会话数据）。总键空间有 218 万个键，占用约 4.9 GB 数据。平均键大小为 2.26 KB。这证实了问题不是单个大键，而是**键的数量极其庞大**——经典的小键洪水问题。

---

## 根因

根因链如下：

```
代码缺陷 → 每次请求创建会话键 → 2 小时内产生 210 万个垃圾键
→ Redis maxmemory 耗尽 → allkeys-lru 逐出循环启动
→ 合法会话被逐出 → 用户被迫重新登录
→ 更多登录请求 → 更多垃圾会话键 → 更多逐出
→ CPU 饱和 (100%) → 所有 Redis 命令变慢（包括 GET）
→ 会话存储完全失效
```

### 缺陷分析

应用代码使用 `uuid.v4()` 作为会话 ID 生成器，**且没有检查该客户端是否已有会话**。每个 HTTP 请求，无论来自新用户还是老用户，都会在 Redis 中生成一个新的会话键：

```javascript
// 有问题的代码模式
app.use(session({
  genid: () => uuid.v4(), // 每次请求生成新键！
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret',
  resave: false,
  saveUninitialized: false  // 本应有所帮助但实际无效
}));
```

`resave: false` 和 `saveUninitialized: false` 本应防止保存未修改的会话。但部署中的中间件顺序问题导致会话中间件在认证之前运行，为所有未认证的请求创建了匿名会话（`session:anon:*`）——包括 API 健康检查、favicon 请求和爬虫流量。

### 为什么逐出策略是错误的

`allkeys-lru` 逐出整个键空间中**最近最少使用**的键。在正常的缓存工作负载下，这是合理的。但对于会话存储：

1. 垃圾键（`session:anon:*`）创建后**再也没有被访问过**——使它们成为 LRU 逐出的首要候选
2. 但 Redis 批量逐出，且逐出算法使用采样，而非全局 LRU
3. 当垃圾键与合法键的比例为 26:1 时，即使随机采样，Redis 也经常只采到垃圾键——从而保留合法会话
4. 然而，持续的写入速率（每秒数千个新键）意味着 Redis **不断在逐出**，消耗大量 CPU 仅用于逐出决策

真正的问题是：`allkeys-lru` 策略加上写入风暴导致了**逐出循环引发的 CPU 饱和**。Redis 忙于逐出，合法命令在逐出工作后面排队等候，导致了数秒的延迟。

### 缺失的保护措施

| 保护措施 | 状态 | 影响 |
|----------|------|------|
| 键 TTL 设置 | 缺失 | 垃圾键永远不会自动过期 |
| Maxmemory 告警 | 缺失 | 团队不知道内存正在耗尽 |
| 键空间监控 | 缺失 | 键数量增长未被发现 |
| 慢查询告警 | 配置不当 | 慢查询已启用但未触发告警 |
| 会话键生命周期审查 | 缺失 | 代码审查未发现缺陷 |
| 连接池限制 | 不足 | 应用服务器打开过多 Redis 连接 |

---

## 解决方案

### 紧急处理：止血

**第 1 步：批量删除垃圾键**

```bash
# 使用 SCAN + 管道批量删除匿名会话键
redis-cli -n 0 --scan --pattern "session:anon:*" | head -500000 | \
  xargs -L 1000 redis-cli DEL

# 重复执行直到垃圾键被清除
redis-cli -n 0 --scan --pattern "session:anon:*" | wc -l  # 检查剩余数量
```

我们使用 `head -500000` 来避免压垮 Redis 连接。每批 1,000 个键（使用 `xargs -L 1000`）在初始时耗时约 3 秒，随着内存压力降低而逐渐加快。

**第 2 步：统计剩余垃圾键**

```bash
# 统计所有会话键
redis-cli KEYS "session:*" | wc -l

# 统计特定模式
redis-cli KEYS "session:anon:*" | wc -l
redis-cli KEYS "session:auth:*" | wc -l
```

> **警告**: 不要在生产的超大键空间中使用 `KEYS`——它会在执行期间阻塞 Redis。我们只在确认实例已经饱和的紧急情况下才使用它。正常操作请使用 `SCAN`。

**第 3 步：临时更改逐出策略**

```bash
# 停止逐出循环
redis-cli CONFIG SET maxmemory-policy noeviction
```

这防止了新写入触发逐出。设置为 `noeviction` 后，超出 `maxmemory` 的写入会被拒绝并返回 OOM 错误——这比无限逐出循环的危害要小。应用程序错误比整个会话存储崩溃要好。

**第 4 步：重启受影响的应用服务器**

删除垃圾键后，我们重启了应用服务器以清除其本地会话缓存并强制建立新的 Redis 连接。

### 短期方案：为会话键设置 TTL

```bash
# 为所有认证会话键设置 TTL（24 小时）
redis-cli -n 0 --scan --pattern "session:auth:*" | \
  xargs -L 1000 redis-cli EXPIRE 86400
```

这确保即使合法会话也会在用户停止使用后最终过期，防止未来的积累。

```bash
# 验证 TTL 已设置
redis-cli INFO keyspace
# 预期: db0:keys=82345,expires=82345,avg_ttl=...
```

### 长期方案：应用程序修复

应用程序代码已修复：

```javascript
// 修复后：使用持久化会话存储和稳定的会话 ID
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret',
  name: 'sid',             // 自定义 cookie 名称
  cookie: { secure: true, httpOnly: true, maxAge: 86400000 },
  resave: false,
  saveUninitialized: false // 不保存空会话
}));

// 会话 ID 由 express-session 在每个会话中自动生成一次，
// 而不是每次请求生成。genid 选项被完全移除。
```

### 监控补充

**内存告警（Prometheus + Alertmanager）：**

```yaml
# prometheus-rules.yml
groups:
  - name: redis
    rules:
      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis 内存使用率 > 80%"
          
      - alert: RedisMemoryCritical
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis 内存使用率 > 95%——即将逐出"
          
      - alert: RedisEvictionsHigh
        expr: rate(redis_evicted_keys_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Redis 每秒逐出超过 10 个键"
```

**慢查询配置：**

```bash
# 配置慢查询日志
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10 ms 阈值
redis-cli CONFIG SET slowlog-max-len 1024           # 保留更多记录
redis-cli SLOWLOG RESET                              # 清除旧记录
```

**键空间监控：**

```bash
# 简单的键数量监控脚本
#!/bin/bash
COUNT=$(redis-cli DBSIZE)
THRESHOLD=500000
if [ "$COUNT" -gt "$THRESHOLD" ]; then
  echo "告警: Redis 键空间有 $COUNT 个键 (阈值: $THRESHOLD)"
  redis-cli INFO keyspace | grep -E "^db0"
fi
```

### 优化 maxmemory-samples

`maxmemory-samples` 设置控制 LRU 样本大小（默认 5）。更大的值能更接近真实的 LRU，但每次逐出的 CPU 开销更大：

```bash
# 默认值（在准确性和 CPU 之间平衡）
redis-cli CONFIG SET maxmemory-samples 5

# 对于逐出敏感的工作负载
redis-cli CONFIG SET maxmemory-samples 10
```

在我们的案例中，调整这个参数不会阻止事故的发生，但值得为未来优化时记录。

### 考虑 Redis Cluster

从长远来看，Redis Cluster 提供了：

- **自动分片**到多个节点（16,384 个哈希槽）
- **分布式内存**——没有单节点 maxmemory 瓶颈
- **高可用性**，支持副本节点和故障转移
- **线性可扩展性**——添加节点即可增加容量

迁移到 Redis Cluster 需要应用程序变更（支持集群的客户端），但对于处理 20K RPM 的会话存储来说，这是一项值得的投资。

---

## 经验教训

### 做得好的

- **监控在部署后几分钟内检测到异常**
- **慢查询日志提供了受影响命令的即时可见性**
- **MEMORY DOCTOR 提供了简洁、可操作的诊断**
- **现有的键清理操作手册可复用**
- **团队及时向利益相关者沟通了事件**

### 做得不好的

1. **没有 maxmemory 告警**: 最大的缺口。内存在不到 2 小时内从 40% 填满到 100%，没有任何通知。
2. **会话键缺少 TTL**: 违反了基本的最佳实践。除非有明确理由，Redis 中的每个键都应该有 TTL。
3. **会话存储使用了不合适的逐出策略**: `allkeys-lru` 是为缓存设计的，不适合会导致用户中断的数据丢失的场景。
4. **代码审查遗漏了会话键缺陷**: `genid: () => uuid.v4()` 模式单独看是无害的。以会话流为重点的审查本可以发现它。
5. **没有键空间增长告警**: 键数量从 8 万增长到 210 万，本应触发自动响应。
6. **慢查询告警配置不当**: 慢查询已启用但日志没有发送到监控系统。它们只能用于手动排查。

### 关键要点

| 领域 | 要点 |
|------|------|
| **TTL 纪律** | 每个 Redis 键都必须设置 TTL。在应用层强制执行，并监控没有 TTL 的键。 |
| **逐出策略** | 会话存储应使用 `volatile-ttl` 或 `volatile-lru`，而不是 `allkeys-*`，以保护数据。 |
| **告警** | 对 `used_memory / maxmemory > 0.8` 和 `evicted_keys > 0` 设置告警。 |
| **容量规划** | 监控键数量增长趋势，而不仅仅是内存使用量。 |
| **代码审查** | 会话生命周期逻辑需要专门的审查关注。 |
| **优雅降级** | `noeviction` 配合应用层 OOM 处理比逐出循环混乱要好。 |

---

## 总结

### 事件时间线

```
14:30  代码部署——每次请求创建会话键开始涌入 Redis
14:45  Redis 内存超过 6 GB (75%)
15:00  内存达到 7.5 GB (94%)，开始逐出
15:05  逐出循环导致 CPU 饱和，慢查询日志填满
15:10  PagerDuty 告警——检测到延迟飙升
15:15  用户报告会话错误、登出
15:20  SRE 开始排查——发现 OOM 错误
15:25  开始紧急键删除
15:40  垃圾键清除完毕，逐出停止
15:45  应用 noeviction 策略，重启应用服务器
16:00  服务恢复正常
16:30  为剩余会话键设置 TTL
17:00  完成监控和告警改进部署
```

### 流程图

```
                    ┌──────────────────────┐
                    │  代码缺陷：每次请求    │
                    │  创建新的会话键        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 2 小时内产生 210 万   │
                    │ 个垃圾键              │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Redis maxmemory       │
                    │ = 8 GB (100%)         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ allkeys-lru 逐出循环  │
                    │ 启动                   │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
            ┌──────────────┐   ┌──────────────────┐
            │ 合法会话被    │   │ CPU 100%——所有   │
            │ 逐出          │   │ 命令变慢(含GET)  │
            └──────┬───────┘   └────────┬─────────┘
                   │                    │
                   ▼                    ▼
            ┌──────────────┐   ┌──────────────────┐
            │ 用户被登出    │   │ 慢查询日志填满    │
            │              │   │ >300ms 命令      │
            └──────┬───────┘   └────────┬─────────┘
                   │                    │
                   └──────┬─────────────┘
                          ▼
               ┌──────────────────────┐
               │ 会话存储完全失效      │
               └──────────────────────┘
```

### 快速参考命令

```bash
# 检查内存
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|maxmemory_policy"

# 检查逐出
redis-cli INFO stats | grep evicted_keys

# 检查慢查询
redis-cli SLOWLOG GET 20

# 分析键空间
redis-cli INFO keyspace
redis-cli MEMORY STATS
redis-cli MEMORY DOCTOR

# 查找键模式
redis-cli --scan --pattern "session:*" | head -100

# 检查客户端
redis-cli CLIENT LIST

# 大键扫描（谨慎使用）
redis-cli --bigkeys

# 批量删除键
redis-cli --scan --pattern "prefix:*" | xargs -L 1000 redis-cli DEL

# 为键模式设置 TTL
redis-cli --scan --pattern "session:*" | xargs -L 1000 redis-cli EXPIRE 86400

# 更改逐出策略（临时）
redis-cli CONFIG SET maxmemory-policy noeviction

# 配置慢查询日志
redis-cli CONFIG SET slowlog-log-slower-than 10000
redis-cli CONFIG SET slowlog-max-len 1024
```

---

*Redis 6.2 * 8 GB maxmemory * 20K RPM * 2 小时 210 万键 * allkeys-lru 逐出循环 * 会话存储故障 *

> 本文是生产故障排查系列的一部分。你是否遇到过 Redis 逐出循环？欢迎在评论区分享你的经历。
