---
title: "Kafka 消费组重平衡异常排查——重平衡风暴如何拖垮实时数据管道"
date: 2026-06-05
weight: 100340
slug: "kafka-consumer-group-rebalance"
tags: ["kafka", "middleware", "troubleshooting", "consumer", "rebalance"]
categories: ["Troubleshooting"]
description: "Kafka 消费组重平衡异常深度排查——session.timeout.ms 和 max.poll.interval.ms 配置不当导致频繁重平衡，实时数据管道从秒级延迟飙升到 15 分钟"
keywords: "kafka 消费组重平衡, kafka rebalance 排查, kafka 消费者超时, kafka session.timeout.ms, kafka max.poll.interval.ms"
draft: false
featured: true
cover:
  image: ""
  caption: "Kafka 消费组重平衡异常排查"
---

# Kafka 消费组重平衡异常排查——重平衡风暴如何拖垮实时数据管道

## 常见搜索关键词

如果你通过以下任一关键词搜索到这里，说明你来对地方了：

- "kafka 消费组重平衡 频繁触发"
- "kafka rebalance 超时 太频繁"
- "kafka 消费延迟 突增 rebalance"
- "kafka session.timeout.ms 太低 重平衡"
- "kafka max.poll.interval.ms 超时"
- "kafka 停止世界 重平衡"
- "kafka 消费组 不稳定 反复重平衡"
- "kafka rebalance 频率 高"

## 故障经过

**环境信息**：Kafka 3.4，单个消费组 12 个消费者，24 个分区，实时分析管道处理用户点击事件。消费者应用部署在 Kubernetes 上，每 Pod 配置 4 vCPU 和 8 GB 内存，使用 Java 17，进行 Avro 反序列化和数据富化处理。

**时间**：周二下午 14:00 高峰期。因促销活动，流量达到正常值的 2 倍。

**症状**：

监控面板在几分钟内全线飘红：

| 指标 | 正常时 | 故障后 |
|------|--------|--------|
| 消费者积压 (Lag) | ~100 条 | **50 万+ 条** |
| 处理延迟 (p99) | 2 秒 | **15 分钟** |
| 重平衡频率 | ~0/小时（稳定） | **30+/小时** |
| 消费组状态 | 稳定 | 持续重平衡中 |

实时数据管道本应在秒级内交付用户点击事件进行分析。然而团队收到的告警显示：分析仪表盘展示的数据已经是 15 分钟前的了。下游的实时机器学习模型接收到了过时的特征数据，推荐质量严重下降。

**影响范围**：
- 实时分析仪表盘延迟 15 分钟以上
- ML 模型因使用过时特征导致服务降级
- 下游计费管道出现数据倾斜
- 值班团队在 14:05 被告警惊醒

## 背景

### Kafka 消费组架构

Kafka 消费组是一组消费者协同消费主题分区的机制。每个分区在同一消费组内只分配给一个消费者，从而实现水平扩展。

```
Topic "click-events" (24 个分区)
  ├── Partition 0  ← 分配给 Consumer A
  ├── Partition 1  ← 分配给 Consumer A
  ├── Partition 2  ← 分配给 Consumer B
  ├── ...
  └── Partition 23 ← 分配给 Consumer L

消费组 "click-analytics-group" (12 个消费者)
  ├── Consumer A ── 负责分区 [0, 1]
  ├── Consumer B ── 负责分区 [2, 3]
  ├── ...
  └── Consumer L ── 负责分区 [22, 23]
```

### 重平衡机制

重平衡是 Kafka 在消费者加入或离开组时重新分配分区的机制。有两种协议：

**Eager 重平衡（默认，`EagerRebalance`）**：组内所有消费者停止消费、撤销所有分区，然后协调者重新分配。在此期间，**没有任何消费者在消费消息**——经典的 stop-the-world。

```
1. Consumer C 崩溃/超时
2. Group Coordinator 检测到故障
3. Coordinator 向 ALL 消费者发送"撤销分区"指令
4. 所有消费者停止处理 → STOP THE WORLD
5. 消费者以新的 generation ID 重新加入
6. Coordinator 重新分配分区
7. 消费者恢复处理
```

**Cooperative 重平衡（`CooperativeRebalance`）**：消费者每次只撤销部分分区，未受影响的消费者继续处理。

### 关键配置参数

| 参数 | 默认值 | 作用 |
|------|--------|------|
| `session.timeout.ms` | 45000 (45s) | 两次心跳最大间隔，超时后 Coordinator 标记消费者死亡 |
| `heartbeat.interval.ms` | 3000 (3s) | 消费者发送心跳的频率 |
| `max.poll.interval.ms` | 300000 (5min) | 两次 poll 之间的最大间隔，超时后标记消费者失败 |
| `max.poll.records` | 500 | 每次 poll 返回的最大记录数 |
| `rebalance.timeout.ms` | 600000 | 消费者重新加入组的超时时间 |

这些超时参数之间形成了一条关键链：

```
session.timeout.ms → 心跳超时检测（快速）
max.poll.interval.ms → 处理超时检测（慢速）

消费者处理 ──> poll() 间隔
     │
     ├── 如果 poll() > max.poll.interval.ms ──> Coordinator 移除消费者 ──> REBALANCE
     │
     └── 如果心跳 > session.timeout.ms ──> Coordinator 标记消费者死亡 ──> REBALANCE
```

## 排查过程

### 第 1 步：检查消费者积压 (Lag)

第一个异常信号是积压曲线飙升。通过 CLI 确认。

```bash
# 检查消费组积压
kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
  --describe --group click-analytics-group
```

```
GROUP                  TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG   CONSUMER-ID     HOST
click-analytics-group  click-events    0          14523000         14523500        500   consumer-1      10.0.0.1
click-analytics-group  click-events    1          14210000         14210500        500   consumer-1      10.0.0.1
click-analytics-group  click-events    2          13890000         14523000       63300  consumer-2      10.0.0.2
click-analytics-group  click-events    3          13950000         14524000       57400  consumer-2      10.0.0.2
...
click-analytics-group  click-events    23         14100000         14525000       42500  consumer-12     10.0.0.12
```

分区 2 和 3 分别积压了 **6.3 万和 5.7 万**，而分区 0 和 1 只有 500。这种不平衡本身就非常可疑——某些消费者明显跟不上。

一分钟后再次查看：

```bash
sleep 60; kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
  --describe --group click-analytics-group | grep -E "LAG|PARTITION" | head -5
```

```
click-analytics-group  click-events    2  13890000  14530000  64000  consumer-2  10.0.0.2
click-analytics-group  click-events    3  13950000  14531000  58100  consumer-2  10.0.0.2
```

积压在**增长**而不是减少——排除了单纯流量突增自动恢复的可能性。

### 第 2 步：检查组成员变化

重平衡表现为消费者反复加入和离开组。

```bash
# 查看组成员详细信息
kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
  --describe --group click-analytics-group --members --verbose
```

```
GROUP                  CONSUMER-ID             HOST          CLIENT-ID             #PARTITIONS
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1            2
click-analytics-group  consumer-2-ghi-jkl      10.0.0.2/1024 consumer-2            2
click-analytics-group  consumer-3-mno-pqr      10.0.0.3/1024 consumer-3            2
...
click-analytics-group  consumer-12-stu-vwx     10.0.0.12/1024 consumer-12          2
```

当前看起来正常，但 Consumer ID 可能已经变了。反复执行以捕捉变化：

```bash
# 每 10 秒查一次，连续 5 次，检测组成员变化
for i in $(seq 1 5); do
  echo "=== 第 $i 次 ==="
  kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
    --describe --group click-analytics-group --members --verbose \
    | grep -E "^click-analytics-group"
  sleep 10
done
```

```
=== 第 1 次 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-ghi-jkl      10.0.0.2/1024 consumer-2  2
...
=== 第 2 次 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-ghi-jkl      10.0.0.2/1024 consumer-2  2
...
=== 第 3 次 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-xyz-rst      10.0.0.2/1024 consumer-2  2   ← CONSUMER ID 变了！
...
=== 第 4 次 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-stu-vwx      10.0.0.2/1024 consumer-2  2   ← 又变了！
...
```

consumer-2 的成员 ID 在 40 秒内变了 3 次。这确认了**频繁重平衡**——consumer-2 反复离开和重新加入组，每次获得新的成员 ID。

### 第 3 步：检查消费者日志中的重平衡事件

接下来检查消费者应用日志。重平衡事件在 `INFO` 级别记录。

```bash
# 搜索重平衡相关日志
kubectl logs -n analytics --selector=app=click-consumer --tail=1000 | grep -i "rebalance\|revoke\|assign\|重平衡\|撤销\|分配"
```

```
2026-06-05 14:02:15 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-2, groupId=click-analytics-group] 
  Revoke previously assigned partitions set(topic=click-events, partition=2, topic=click-events, partition=3)
2026-06-05 14:02:15 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-2, groupId=click-analytics-group] 
  组成员资格因心跳超时而取消: consumer-2-ghi-jkl 已离开组, 重新加入中
2026-06-05 14:02:16 WARN  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-2, groupId=click-analytics-group] 
  自动提交偏移量失败，因为该组成员资格已失效。回退到重置重试逻辑。
2026-06-05 14:02:18 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-2, groupId=click-analytics-group] 
  加入组并完成分配: 分区=[click-events-2, click-events-3]
2026-06-05 14:02:28 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-2, groupId=click-analytics-group] 
  Revoke previously assigned partitions set(topic=click-events, partition=2, topic=click-events, partition=3)
...
```

日志每 30-60 秒重复一次。关键发现：
- 心跳超时导致组成员资格被撤销
- 消费者重新加入并获得分配
- 循环反复

其他消费者的日志：

```bash
kubectl logs -n analytics click-consumer-0 --tail=500 | grep -i "rebalance\|revoke"
```

```
2026-06-05 14:02:15 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-1, groupId=click-analytics-group] 
  Revoke previously assigned partitions set(..., ..., ...)
2026-06-05 14:02:18 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-1, groupId=click-analytics-group] 
  Assigned to partition(s): ...
```

每次 consumer-2 掉线，所有消费者都被迫撤销分区并重新加入。这就是 **eager rebalance 的 stop-the-world 行为**。

### 第 4 步：JMX 指标——重平衡频率

通过 JMX 指标获取精确的重平衡计数。

```bash
# 如果使用 Prometheus JMX exporter:
curl -s http://localhost:8080/metrics | grep -i rebalance
```

```
# HELP kafka_consumer_coordinator_rebalance_rate_per_hour 每小时重平衡次数
# TYPE kafka_consumer_coordinator_rebalance_rate_per_hour gauge
kafka_consumer_coordinator_rebalance_rate_per_hour{client_id="consumer-1",} 34.0
kafka_consumer_coordinator_rebalance_rate_per_hour{client_id="consumer-2",} 34.0
...

# HELP kafka_consumer_coordinator_rebalance_total 重平衡总次数
# TYPE kafka_consumer_coordinator_rebalance_total counter
kafka_consumer_coordinator_rebalance_total{client_id="consumer-1",} 127.0
kafka_consumer_coordinator_rebalance_total{client_id="consumer-2",} 127.0
```

**每小时 34 次重平衡**——大约每 2 分钟一次。正常值是 0。

### 第 5 步：检查消费者配置

确定问题是重平衡后，下一步：*为什么* consumer-2 被踢出？

```bash
# 查看消费者配置
kubectl exec -n analytics deploy/click-consumer-2 -- cat /app/config/application.yml | grep -A 20 "kafka"
```

```yaml
spring:
  kafka:
    consumer:
      bootstrap-servers: kafka-cluster:9092
      group-id: click-analytics-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: io.confluent.kafka.serializers.KafkaAvroDeserializer
      properties:
        session.timeout.ms: 10000        # 10 秒 ← 异常低
        heartbeat.interval.ms: 3000      # 3 秒
        max.poll.interval.ms: 300000     # 5 分钟
        max.poll.records: 500            # 每次 poll 500 条 ← 太高
        max.partition.fetch.bytes: 1048576 # 1 MB
```

三个红旗：

| 配置项 | 当前值 | Kafka 推荐值 | 问题 |
|--------|--------|-------------|------|
| `session.timeout.ms` | 10,000 (10s) | 45,000 (45s) | 太低——JVM GC 暂停轻松超过 10s |
| `max.poll.interval.ms` | 300,000 (5min) | 取决于 workload | 500 条 Avro 记录含富化处理需要 6-8 分钟 |
| `max.poll.records` | 500 | 取决于处理时间 | 对于富化逻辑来说，每次拉取太多 |

## 根因分析

排查揭示了一个级联故障链，将一次简单的流量突增变成了全面故障。以下是按顺序列出的根因：

### 1. `session.timeout.ms=10000`——误报心跳超时

`session.timeout.ms` 被设置为 10 秒（旧版 Spring Kafka 的默认值），而 Kafka 3.4 推荐的最小值是 45 秒。

消费者应用运行在 Kubernetes 上，4 vCPU + 8 GB 内存。高峰期 JVM 的 **Young GC 暂停 2-5 秒**，处理大 Avro 记录时偶尔 **Full GC 暂停 8-12 秒**。

10 秒的超时窗口意味着一次 11 秒的 Full GC 暂停就足以让消费者错过心跳窗口。Group Coordinator 会标记该消费者死亡，触发重平衡。

```
正常情况:    心跳 (3s) → 心跳 (3s) → 心跳 (3s) → OK
GC 期间:    心跳 (3s) → [GC 暂停 11s] → ❌ session.timeout.ms=10s 超时
                                     ↓
            Coordinator 标记消费者为 DEAD
                                     ↓
                                 重平衡
```

### 2. `max.poll.interval.ms=300000`——批处理超时

即使心跳成功，`max.poll.interval.ms` 设置为 5 分钟（300,000 ms），配合 `max.poll.records=500`，每次拉取最多 500 条记录。每条记录的富化逻辑：

1. Avro 反序列化（5-10 ms）
2. 从 Redis 获取用户画像（20-50 ms）
3. 获取地理 IP 数据（10-30 ms）
4. 写入下游主题（5 ms）

每条记录总处理时间：**40-95 ms**。500 条 = **20-47 秒**（正常负载下）。

但 **2 倍流量突发**下，Redis 延迟上升（缓存未命中率增加），单条记录处理时间膨胀到 **200-500 ms**。500 条 = **100-250 秒**。

250 秒是 4.2 分钟——在 5 分钟轮询间隔之内。但偶尔的异常（Redis 连接重试、网络抖动）将批处理时间推到 **6-8 分钟**，超过了 300,000 ms 限制。

当 `max.poll.interval.ms` 被超过时，Kafka 认为消费者卡死并标记为失败——触发又一次重平衡。

```
正常批处理 (500 条): 20-47 秒 → 在 5 分钟限制内 ✓
高峰批处理 (500 条): 100-250 秒 → 勉强在限制内
高峰 + 网络抖动:     360-480 秒 → 超过 300s 限制 → 重平衡
```

### 3. `max.poll.records=500`——放大因素

500 条/次对于快速处理来说是合理的，但富化逻辑的延迟方差很大，这制造了高风险。降低这个值可以：
- 减少每批处理时间
- 降低超过 `max.poll.interval.ms` 的概率
- 减少 GC 压力（每批次内存中记录更少）

### 4. Eager 重平衡协议——Stop-the-World 放大器

使用默认的 eager 重平衡协议，每次 consumer-2 被踢出（由于会话超时或轮询超时），**全部 12 个消费者** 都停止处理、撤销分区并重新加入。这造成了级联：

1. Consumer-2 超时 → 触发重平衡
2. 全部 12 个消费者停止处理 → 所有分区的积压开始增长
3. 重新分配后恢复处理，但积压已经增大
4. Consumer-2 处理积压 → 耗时更长 → 超过 `max.poll.interval.ms` → 再次重平衡
5. 重复——每次重平衡都会增加处理积压，增加处理时间，增加超时概率

```
流量突增 → Consumer-2 处理变慢
         ↓
    session.timeout.ms 超时 (GC 暂停)
         ↓
    Eager 重平衡 (所有消费者停止)
         ↓
    所有分区积压增长
         ↓
    消费者追赶积压
         ↓
    max.poll.interval.ms 超时 (积压过多)
         ↓
    又一轮重平衡 → 更多积压 → 死亡螺旋
```

这就是 **重平衡死亡螺旋**——重平衡引起更多的重平衡。

## 解决方案

### 紧急修复（止血）

首要任务是稳定消费组。我们调整了运行中消费者的三个配置参数。

```bash
# 通过环境变量紧急更新配置
kubectl set env deployment/click-consumer \
  SPRING_KAFKA_CONSUMER_PROPERTIES_SESSION_TIMEOUT_MS=45000 \
  SPRING_KAFKA_CONSUMER_PROPERTIES_MAX_POLL_INTERVAL_MS=600000 \
  SPRING_KAFKA_CONSUMER_PROPERTIES_MAX_POLL_RECORDS=200
```

```bash
# 滚动重启以生效
kubectl rollout restart deployment/click-consumer
```

验证滚动升级：

```bash
kubectl rollout status deployment/click-consumer
```

```
deployment "click-consumer" successfully rolled out
```

**配置对比（修复前后）**：

| 参数 | 修复前 | 修复后 | 原因 |
|------|--------|--------|------|
| `session.timeout.ms` | 10,000 (10s) | **45,000 (45s)** | 容纳最长 30 秒的 GC 暂停 |
| `heartbeat.interval.ms` | 3,000 (3s) | **3,000 (3s)** | 不变——保持快速心跳检测 |
| `max.poll.interval.ms` | 300,000 (5min) | **600,000 (10min)** | 给消费者足够时间处理大批次 |
| `max.poll.records` | 500 | **200** | 减少每批处理时间和内存压力 |

### 验证恢复

```bash
# 检查稳定后的积压
kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
  --describe --group click-analytics-group
```

```
GROUP                  TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
click-analytics-group  click-events    0          14523500         14524000        500
click-analytics-group  click-events    1          14523500         14524000        500
...
click-analytics-group  click-events    23         14524000         14524500        500
```

积压稳定并开始消化。重平衡频率降到 0。

```bash
curl -s http://localhost:8080/metrics | grep "rebalance_rate_per_hour"
```

```
kafka_consumer_coordinator_rebalance_rate_per_hour{client_id="consumer-1",} 0.0
```

### 长期方案：切换到 Cooperative 重平衡

Eager 重平衡协议放大了问题。我们切换到增量协同重平衡，只撤销受影响消费者的分区。

```yaml
# application.yml
spring:
  kafka:
    consumer:
      properties:
        # ... 其他配置
        group.protocol: consumer       # Kafka 3.4+ 现代组协议
        partition.assignment.strategy:
          - org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

等效写法：

```yaml
spring:
  kafka:
    consumer:
      properties:
        rebalance.protocol: cooperative
```

**为什么 Cooperative 重平衡有效**：

| 方面 | Eager（修复前） | Cooperative（修复后） |
|------|----------------|----------------------|
| 行为 | 所有消费者停止，撤销所有，重新分配 | 仅撤销受影响的分区 |
| 对健康消费者的影响 | 完全停止处理 | 继续处理未受影响的分区 |
| 稳定所需时间 | 长（全部重新加入） | 短（最小化中断） |
| 放大级联故障？ | 是（每次重平衡影响所有人） | 否（隔离到受影响的消费者） |

### 添加监控

我们添加了以下告警，以便在下一次演变成事故之前捕获：

| 告警 | 阈值 | 动作 |
|------|------|------|
| 重平衡频率 | 任何消费组 > 5/小时 | 呼叫值班 |
| 消费者积压 | 任意分区 > 10,000 | 呼叫值班 |
| `max.poll.interval.ms` 超时 | 日志中出现 | 告警提醒 |
| 会话超时 | 日志中出现 | 告警提醒 |
| GC 暂停时间 | > 5s (p99) | 排查 |

### 配置检查清单

将 Kafka 消费者部署到生产环境前，确认以下配置：

- [ ] `session.timeout.ms` >= 45000（或 3 倍预期最大 GC 暂停时间）
- [ ] `max.poll.interval.ms` >= 2 倍预期最大批处理时间
- [ ] `max.poll.records` 根据处理逻辑设定合理的批次大小
- [ ] 心跳间隔 `heartbeat.interval.ms` <= `session.timeout.ms` 的 1/3
- [ ] 优先使用 Cooperative 重平衡协议
- [ ] 监控重平衡频率和消费者积压
- [ ] 开启 JVM GC 日志，便于关联 GC 暂停和重平衡事件

## 经验教训

1. **默认配置不等于安全配置。** Spring Kafka 的 `session.timeout.ms=10000` 在开发环境也许没问题，但在有 GC 暂停的生产环境中就是定时炸弹。务必根据工作负载的最差处理时间来调整消费者超时，而不是平均值。

2. **重平衡会引发更多重平衡。** Eager 重平衡的 stop-the-world 特性意味着一个慢消费者就能拖垮整个组。重平衡期间产生的积压使消费者在重新分配后更容易超时，形成死亡螺旋。

3. **`max.poll.records` 是控制一切的杠杆。** 与其围绕大批次调整超时，不如先减小批次大小。200 条/次通常比 500 条更合理。它降低了单批处理的延迟方差、GC 压力和超过 `max.poll.interval.ms` 的风险。

4. **GC 暂停和 Kafka 超时是一对危险组合。** 如果消费者做任何非平凡的处理（反序列化、富化、外部调用），务必监控 GC 暂停时间，确保 `session.timeout.ms` 能容纳最坏情况。10 秒超时的 JVM 却可能发生 8 秒的 Full GC 暂停——这就是灾难的配方。

5. **2026 年的今天，Cooperative 重平衡应该成为默认选择。** Kafka 3.4+ 已经支持 cooperative 重平衡多年。对现代工作负载来说，几乎没有任何理由继续使用 eager 重平衡。增量方式将故障隔离到受影响的消费者，而不是污染整个组。

6. **始终将重平衡频率作为一等信号来监控。** 如果你的 Kafka 仪表盘上没有"每小时重平衡次数"这个指标，加上它。非零的重平衡频率几乎总是故障的信号。零重平衡才是唯一可接受的稳态。

## 总结

### 故障链路

```
配置不当的消费者
  │
  ├── session.timeout.ms = 10s（太低）
  ├── max.poll.interval.ms = 5min（高峰时不够）
  └── max.poll.records = 500（太高）
            │
            ▼
    流量突增（2 倍正常值）
            │
            ▼
    JVM GC 暂停超过 10s
            │
            ▼
    session.timeout.ms 超时 → 消费者被标记死亡
            │
            ▼
    EAGER 重平衡（全部 12 个消费者停止）
            │
            ▼
    所有分区的积压开始增长
            │
            ▼
    消费者追赶积压 → 超过 max.poll.interval.ms
            │
            ▼
    又一轮重平衡 → 积压进一步增长
            │
            ▼
    死亡螺旋：重平衡 → 积压 → 超时 → 重平衡
            │
            ▼
    管道延迟：2 秒 → 15 分钟
    消费者积压：100 → 50 万+
```

### 恢复流程

```
紧急配置变更
  │
  ├── session.timeout.ms: 10s → 45s
  ├── max.poll.interval.ms: 5min → 10min
  └── max.poll.records: 500 → 200
            │
            ▼
    消费者滚动重启
            │
            ▼
    重平衡频率降到 0
            │
            ▼
    积压消化：50 万 → 500
            │
            ▼
    处理延迟：15 分钟 → 2 秒
            │
            ▼
    长期方案：cooperative 重平衡 + 监控
```

**核心总结**：三个错误的配置值加上一次流量突发，创造了一个自持的重平衡死亡螺旋，拖垮了实时数据管道。一旦理解了故障链，修复就非常直接——调整超时以容纳真实世界的处理波动，减小批次大小，切换到 cooperative 重平衡以隔离故障。
