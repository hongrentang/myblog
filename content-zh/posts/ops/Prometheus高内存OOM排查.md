---
title: "Prometheus 高内存 OOM 排查实录——当监控系统成为事故本身"
date: 2026-06-07
weight: 100370
slug: "prometheus-oom-high-memory"
tags: ["prometheus", "monitoring", "troubleshooting", "oom", "performance"]
categories: ["Troubleshooting"]
description: "一次 Prometheus OOM 事故——一个携带高基数标签的指标和过度宽泛的聚合 Recording Rule 如何让 Prometheus Pod 内存耗尽，导致整个监控栈瘫痪"
keywords: "prometheus oom, prometheus 高内存, prometheus tsdb, prometheus 标签基数, prometheus recording rules 优化"
draft: false
featured: true
cover:
  image: ""
  caption: "Prometheus OOM 排查实录"
---

# Prometheus 高内存 OOM 排查实录——当监控系统成为事故本身

## 常见搜索查询

| 搜索关键词 | 意图 |
|---|---|
| prometheus oom | Prometheus 因内存不足而崩溃 |
| prometheus 内存占用高 | 排查 Prometheus 内存泄漏或膨胀 |
| prometheus tsdb 内存 | 定位 TSDB 内存消耗 |
| prometheus 标签基数过高 | 怀疑高基数标签导致问题 |
| prometheus recording rule 优化 | 减少 Recording Rule 的内存开销 |
| prometheus OOMKilled kubernetes | Prometheus 在 K8s 容器中被 OOM 杀死 |
| promtool tsdb analyze 分析基数 | 使用 promtool 诊断序列爆炸 |
| prometheus wal replay 内存 | WAL 回放导致的内存飙升 |
| prometheus heap profile | 对 Prometheus 进行 Heap 性能分析 |
| prometheus /api/v1/status/tsdb | 通过 HTTP API 查看标签序列数 |

---

## 故障经过

**环境信息：**

| 项目 | 详情 |
|---|---|
| Prometheus 版本 | v2.45 |
| 部署方式 | Kubernetes (StatefulSet) |
| 内存限制 | 8 GB |
| 采集目标 | ~500 个 |
| 时间序列数 | ~100 万 |
| 数据保留 | 15 天 |
| 采集间隔 | 15 秒 |

一个平静的周二下午。一个新的微服务——"订单分析服务"——上线了。标准的操作流程：添加 ServiceMonitor，让 Prometheus 自动发现并开始采集指标。

三小时后，PagerDuty 开始疯狂告警。

**症状链：**

```
15:42  — Grafana 仪表盘全部变灰（"No data"）
15:43  — Alertmanager 触发 "Watchdog" 告警（心跳丢失）
15:44  — Watchdog 升级为 "AlertmanagerDown"
15:45  — PagerDuty 通知整个 SRE 团队
```

Prometheus Pod 挂了。然后它又挂了。再挂一次。每 3-4 小时一次，像闹钟一样准时。每次重启之间有一段短暂的窗口期：Prometheus 启动、开始采集、摄入数据几个小时——然后再次宕机。

整个监控栈完全失明。没有告警触发、没有仪表盘渲染、没有可用的指标数据来辅助排查。监控系统本身，成了事故的主角。

---

## 背景

要理解 Prometheus 为什么会内存耗尽，需要先了解它的内存模型。

### Prometheus 内存模型

| 组件 | 作用 | 内存影响 |
|---|---|---|
| **TSDB**（时序数据库） | 将时序数据以 mmap 方式落盘存储 | 中等——大部分由磁盘承载 |
| **WAL**（预写日志） | 写入压缩前的临时缓冲区 | 写入量大时内存占用高 |
| **内存 Chunk** | 每个序列最新的样本暂存在内存中（尚未持久化） | **高**——与序列数成正比 |
| **Head block** | 接收写入的"活跃"TSDB 块 | 高——所有序列的当前 Chunk 都在这里 |
| **查询引擎** | 执行 PromQL 查询 | 突发高——复杂查询在计算过程中消耗大量内存 |

关键要点：**Prometheus 为每个活跃采集的序列在内存中维护一个 Chunk**（15 秒采集间隔下，约 120 个样本 = 30 分钟数据）。如果有 100 万个序列，那就是 100 万个 Chunk 在内存里。每个 Chunk 大约 1-2 KB。数学账很快就会算不过来。

### 什么是标签基数

基数 = 一个指标所有唯一标签组合的数量。

```
http_request_duration_ms{user_id="u1", endpoint="/api/checkout", method="POST", status_code="200", instance="pod-1"}
http_request_duration_ms{user_id="u2", endpoint="/api/checkout", method="POST", status_code="200", instance="pod-1"}
...
```

如果一个 `user_id` 标签有 10,000 个唯一值，同时有 50 个端点、5 个方法、10 个状态码、500 个实例——那么总序列数就是：

```
10,000 (用户) × 50 (端点) × 5 (方法) × 10 (状态码) × 500 (实例)
= 12,500,000,000,000 条序列
```

实际上 Prometheus 会做去重，但即使按保守估算，引入 `user_id` 这样的高基数标签也能把序列数放大 1,000 倍甚至更多。

### Recording Rule

Recording Rule 预计算 PromQL 表达式并将结果存为新的时间序列。它们对加速仪表盘很有用，但也会**额外消耗内存**——每条 Recording Rule 的结果都是一条新的时间序列，需要自己的内存 Chunk、WAL 条目和存储块。

---

## 排查过程

当我们介入时，Prometheus Pod 已经重启了 4 次。我们必须快速行动——监控窗口正在缩短。

### 第 1 步：检查 Prometheus Pod 日志

```bash
# 检查 Pod 死亡原因
kubectl describe pod prometheus-0 | grep -A5 "Last State"
```

```
Last State: Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Restart Count: 4
```

Exit code 137 = SIGKILL = OOMKilled。内核的 OOM killer 终止了 Prometheus 进程，因为容器超过了 8 GB 的内存限制。

```bash
# 查看最近日志寻找线索
kubectl logs prometheus-0 --tail=50
```

```
ts=2026-06-07T15:42:00.123Z caller=scrape.go:1543 level=warn
  msg="Error scraping target" target="order-analytics/0.0.0.0:8080"
  err="context deadline exceeded"
ts=2026-06-07T15:42:01.456Z caller=scrape.go:1543 level=warn
  msg="Error scraping target" target="order-analytics/0.0.0.0:8080"
  err="context deadline exceeded"
...
ts=2026-06-07T15:42:30.789Z caller=compact.go:516 level=error
  msg="error compacting WAL" err="no such process"
```

采集开始超时，然后压缩失败——进程已经在被杀的过程中了。

### 第 2 步：查看崩溃前的内存指标

我们需要看内存趋势。由于 Prometheus 本身挂了，我们依靠 kube-state-metrics 和 node exporter 的数据（存在另一套 Prometheus 里——监控监控系统，有点讽刺）。

```bash
# 从另一套监控实例查询
container_memory_working_set_bytes{pod="prometheus-0", namespace="monitoring"}
```

```
12:00 — 3.2 GB
13:00 — 4.8 GB
14:00 — 6.1 GB
15:00 — 7.6 GB
15:42 — OOMKilled（达到 8 GB 限制）
```

平稳的线性上升。不是内存泄漏——只是序列太多，数据产生的速度超过了 Prometheus 压缩和落盘的速度。

### 第 3 步：用 promtool 分析 TSDB

在启动了一个使用精简采集配置的备用 Prometheus 实例后，我们在 TSDB 数据目录（已保存到持久卷）上执行了基数分析。

```bash
# 分析 TSDB 基数
kubectl exec prometheus-0 -- promtool tsdb analyze /prometheus
```

```
Block ID: 01J3XYZ...
  Duration: 2h
  Series: 1,234,567
  ...
  
Label names with highest cardinality:
  user_id:      10,420 unique values  ←  就是这个
  instance:        512 unique values
  endpoint:         56 unique values
  status_code:      34 unique values
  method:            8 unique values
```

`user_id` 有 10,420 个唯一值。结合其他标签，总序列数超过了 120 万。

```bash
# 查看每个指标的序列数
promtool tsdb analyze /prometheus --extended
```

```
Metric with highest number of series:
  http_request_duration_ms:        1,150,000 series  ←  就是这个
  http_request_duration_ms_sum:      920,000 series
  http_request_duration_ms_count:    920,000 series
  prometheus_tsdb_head_series:             1  (元指标)
  ...
```

仅 `http_request_duration_ms` 这一个指标家族就占了 115 万条序列——数据库中所有序列的 90% 以上。

### 第 4 步：通过 API 查看标签基数

```bash
# 查看标签名称及其序列数
kubectl exec prometheus-0 -- wget -qO- http://localhost:9090/api/v1/status/tsdb
```

```json
{
  "status": "success",
  "data": {
    "seriesCountByMetricName": {
      "http_request_duration_ms": 1150000,
      "prometheus_tsdb_head_series": 1
    },
    "labelValueCountByLabelName": {
      "user_id": 10420,
      "instance": 512,
      "endpoint": 56,
      ...
    },
    "memoryInBytesByLabelName": {
      "user_id": 892000000,
      ...
    }
  }
}
```

`memoryInBytesByLabelName` 字段很有说服力：仅 `user_id` 标签就消耗了约 892 MB 内存（标签索引和 postings）。

```bash
# 查看活跃序列数
kubectl exec prometheus-0 -- wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series'
```

```
{
  "status": "success",
  "data": {
    "result": [{"value": [1717758000, "1234567"]}]
  }
}
```

Head block 中有 1,234,567 个活跃序列。在 8 GB 内存限制下，每个序列大约只有 **6.5 KB** 的预算——非常紧张，在 WAL 回放或查询突发时很容易被突破。

### 第 5 步：检查 Recording Rule

```bash
# 列出 recording rule
kubectl exec prometheus-0 -- cat /etc/prometheus/rules/recording-rules.yml
```

```yaml
groups:
  - name: http_request_recording_rules
    rules:
      - record: "http_request_duration_ms_avg_5m"
        expr: |
          rate(http_request_duration_ms[5m])
      - record: "http_request_duration_ms_p99_5m"
        expr: |
          histogram_quantile(0.99, rate(http_request_duration_ms_bucket[5m]))
```

初看这些规则人畜无害。但 `rate(http_request_duration_ms[5m])` **对所有标签维度**进行了聚合——包括 `user_id`。这意味着：

- Recording Rule 为源指标的**每一种标签组合**生成一条新的时间序列
- 在 115 万条源序列的基础上，Recording Rule 又产生了 115 万条新序列
- 序列数翻倍 = 内存 Chunk 翻倍 = 内存压力翻倍

问题不只在指标本身——Recording Rule 将高基数问题复制（至少是翻倍）了。

### 第 6 步：查看序列数和存储块

```bash
# 统计磁盘上的块和序列数
kubectl exec prometheus-0 -- promtool tsdb list /prometheus
```

```
BLOCK ULID                  MIN TIME       MAX TIME       DURATION    NUM SAMPLES  NUM CHUNKS  NUM SERIES  SIZE
01J3XYZ...                  2026-06-07-12  2026-06-07-14  2h          850,000,000  1,200,000   1,150,000   4.2 GB
01J3ZAB...                  2026-06-07-10  2026-06-07-12  2h          780,000,000  1,100,000   1,050,000   3.9 GB
...
```

每个 2 小时的块在磁盘上有 4+ GB。压缩时 Prometheus 会将块加载到内存——多个块同时压缩时，内存使用可能飙升 2-3 倍。

---

## 根因

| 层面 | 原因 |
|---|---|
| 直接原因 | 容器超过 8 GB 内存限制 → OOMKilled |
| 触发条件 | 新的 `order-analytics` 服务暴露了带有高基数 `user_id` 标签（10K+ 唯一值）的 `http_request_duration_ms` 指标 |
| 序列爆炸 | `http_request_duration_ms` 从约 5 万条序列一夜之间增长到 **115 万条**——**23 倍** |
| 内存放大 | Recording Rule 跨所有维度聚合，将内存中的有效序列数**翻倍** |
| 恶性循环 | 每次 OOM 重启后，WAL 回放将所有近期样本同时加载到内存——导致**内存尖峰**，每轮循环都更快撞到限制 |
| 为什么是 3-4 小时 | Prometheus 需要足够的时间摄入新序列 + 执行压缩 + 达到内存上限 |

### 具体数据

```
部署之前：
  序列数: ~50,000
  内存使用: ~2.5 GB
  Recording Rule: 5 条规则作用于 ~5 万条序列
  稳定性: 30+ 天正常运行

部署之后：
  新指标: http_request_duration_ms{user_id, request_id, endpoint, method, status_code, instance}
  user_id 基数: 1 万个唯一值
  新增序列: 115 万（500 实例 × 50 端点 × 10 方法 × 5 状态码 × 1 万用户——去重后）
  内存使用: 7.6 GB 且持续攀升
  Recording Rule: 扩展后作用于 115 万条序列
  稳定性: 3-4 小时 OOM 一次
```

### WAL 回放死亡螺旋

一个关键的放大效应让情况更糟：

1. Prometheus OOM → 进程退出
2. Pod 重启 → Prometheus 回放 WAL 重建内存状态
3. WAL 回放将所有**近期未压缩的样本**一次性加载到内存
4. 回放期间内存飙升至约 6 GB（甚至还没开始正常采集）
5. 正常采集开始，加入更多序列
6. 内存超过 8 GB → 再次 OOM
7. 循环，WAL 每轮越来越大（因为块从未完全压缩）

这就是 Prometheus OOM 的"死亡螺旋"：**每次重启都让下一次来得更快**，因为 WAL 在不断增长。

---

## 解决方案

### 紧急措施：增加内存限制（立即缓解）

最快止血方式——为真正修复争取时间。

```bash
kubectl edit statefulset prometheus -n monitoring
```

```yaml
resources:
  limits:
    memory: 16Gi       # 原来是 8Gi
  requests:
    memory: 12Gi       # 原来是 6Gi
```

```bash
# 生效并等待重启完成
kubectl rollout status statefulset prometheus -n monitoring
```

这立即停止了 OOM 循环。然而，不解决根因而单纯增加内存只是拖延问题——而且会增加成本。

### 通过 Relabel 配置丢弃高基数标签

真正的修复：从问题指标中丢弃 `user_id` 和 `request_id` 标签。这些标签对按请求调试很有用，但**绝不应该存储在 Prometheus TSDB 中**。它们属于日志系统（Elasticsearch、Loki）或链路追踪后端（Jaeger、Tempo），而不是时序数据库。

```yaml
# prometheus.yml — relabel_configs
scrape_configs:
  - job_name: "order-analytics"
    ...
    relabel_configs:
      # 丢弃导致 TSDB 崩溃的高基数标签
      - source_labels: [__name__]
        regex: "http_request_duration_ms.*"
        action: labeldrop
        target_label: user_id
      
      # request_id 同样处理——相同的问题
      - source_labels: [__name__]
        regex: "http_request_duration_ms.*"
        action: labeldrop
        target_label: request_id
```

或者在全局层面丢弃，以捕获任何可能携带类似标签的指标：

```yaml
# 全局 relabel——捕获所有包含问题标签的指标
relabel_configs:
  - action: labeldrop
    regex: "(user_id|request_id|session_id|transaction_id)"
```

**重要说明**：`labeldrop` 丢弃标签但保留指标。指标仍然有 `endpoint`、`method`、`status_code`、`instance`——这些对于有用的聚合已经足够了。你会失去按用户的粒度，但能换来一个正常工作的监控系统。

应用配置并重启后：

```
序列数: 115 万 → ~2.5 万（减少 98%）
内存使用: 7.6 GB → ~1.8 GB
```

### 优化 Recording Rule

Recording Rule 之前对所有维度（包括高基数标签）进行了全量聚合。丢弃这些标签后，规则变得便宜了很多。但还有一些额外的最佳实践：

```yaml
# 优化前（昂贵——跨所有维度聚合，包括 user_id）
groups:
  - name: http_request_recording_rules
    rules:
      - record: "http_request_duration_ms_avg_5m"
        expr: |
          rate(http_request_duration_ms[5m])

# 优化后（更便宜——仅按有用的维度聚合）
groups:
  - name: http_request_recording_rules_aggregated
    rules:
      - record: "job:http_request_duration_ms:rate5m"
        expr: |
          sum by (job, endpoint, method) (rate(http_request_duration_ms[5m]))
      
      - record: "job:http_request_duration_ms:p99_5m"
        expr: |
          histogram_quantile(0.99,
            sum by (job, endpoint, method, le) (rate(http_request_duration_ms_bucket[5m])))
```

关键优化点：
- **尽早聚合**：在 Recording 之前使用 `sum by (important_labels)`。Recording Rule 结果中的序列数越少 = 内存越少。
- **限制维度**：只包含仪表盘实际会用到的标签。
- **避免无参 `by()`**：如果你写 `rate(http_request_duration_ms[5m])` 而不带聚合函数，每种唯一的标签组合都会在 Recording Rule 输出中生成一条序列——复刻源指标的基数。

### 调整 TSDB 保留时间和块大小

```bash
# 缩短保留时间（如果 15 天不是硬需求）
--storage.tsdb.retention.time=7d

# 控制最大块大小，减少压缩时的内存压力
--storage.tsdb.max-block-duration=2h

# 最小块大小（更小的块 = 更多压缩但每次压缩内存更少）
--storage.tsdb.min-block-duration=30m
```

```yaml
# 在 StatefulSet args 中设置
args:
  - "--storage.tsdb.retention.time=7d"
  - "--storage.tsdb.max-block-duration=2h"
  - "--storage.tsdb.min-block-duration=30m"
```

### 验证恢复

```bash
# 1. 确认不再有 OOM 杀死
kubectl describe pod prometheus-0 | grep -A5 "Last State"
# 应显示 "Running" 且没有近期 OOM

# 2. 检查内存使用
kubectl top pod prometheus-0 -n monitoring
# 应稳定在 4 GB 以下

# 3. 确认序列数受控
kubectl exec prometheus-0 -- wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series'
# 应显示大幅减少的序列数

# 4. 确认仪表盘正常工作
curl -s http://prometheus:9090/api/v1/query?query=http_request_duration_ms_count | jq '.data.result | length'
# 应返回可控的序列数
```

### 长期方案：考虑 Thanos 或 Cortex

对于确实需要长期保留高基数数据的组织，可以考虑多层架构：

| 方案 | 适用场景 |
|---|---|
| **Thanos** | Sidecar 模式 + 对象存储；适合想要"无限制 Prometheus"的场景 |
| **Cortex / Mimir** | 多租户、水平扩展；适合大规模部署 |
| **VictoriaMetrics** | 即插即用的 Prometheus 兼容替代品，内存效率更高 |
| **日志系统存储高基数标签** | `user_id` 和 `request_id` 属于日志/链路追踪，不属于指标 |

---

## 经验教训

1. **基数问题是 Prometheus 的头号隐藏杀手。** 一个携带高基数标签的指标就像内存泄漏——缓慢、累积、灾难性的。新指标上线前一定要审计。`promtool tsdb analyze` 应该成为 CI/CD 流水线的一部分。

2. **绝不要把无界标签放入 Prometheus。** `user_id`、`request_id`、`session_id`、`email`、`ip_address`——任何有无穷或用户自定义取值的标签——都不应该放在时序数据库里。Prometheus 官方文档明确警告过这一点，忽视它就会直接导致 OOM。

   > ⚠️ **Prometheus 文档警告**："请注意标签的基数。一个有很多不同取值的标签会导致 Prometheus 使用大量内存。"

3. **Recording Rule 会放大基数问题。** 如果你的源指标基数很高，一个跨所有维度聚合的 Recording Rule 会在输出中产生同样高的基数。在 Recording 之前始终使用 `sum by (important_labels)` 来降低维度。

4. **监控监控系统不是可选项。** Prometheus 挂了，你就瞎了。设置一套独立的轻量级 Prometheus（或使用托管服务）来监控主 Prometheus——内存使用、序列数、采集失败情况。没有它，你就是在零指标的情况下调试 OOM，几乎不可能。

5. **WAL 回放是死亡螺旋。** 如果 Prometheus 挂了一次，下一次会挂得更快，因为每轮未完成的循环都在让 WAL 增长。打破这个循环要么需要（a）在重启前大幅减少采集负载，要么（b）增加内存以撑过回放。

6. **8 GB 对 100 万序列来说太紧了。** 一个粗略的经验法则：每 100 万序列大约消耗 4-6 GB 内存（仅 head block），还要加上 WAL、查询和压缩的开销。每 100 万序列至少留 8-10 GB——这还没算 Recording Rule。

---

## 总结

| 阶段 | 关键动作 | 结果 |
|---|---|---|
| 发现 | Pod OOMKilled，PagerDuty 升级 | 事故宣告 |
| 分类 | `kubectl describe pod` → OOMKilled | 锁定排查方向 |
| 诊断 | `promtool tsdb analyze` → 来自 `http_request_duration_ms` 的 115 万序列 + `user_id` 标签 | 根因定位 |
| 紧急修复 | 内存从 8 Gi 增加到 16 Gi | OOM 立即停止 |
| 根本修复 | 通过 relabel_config 丢弃 `user_id` 和 `request_id` | 序列数减少 98% |
| 优化 | 使用显式 `by()` 聚合重写 Recording Rule | 内存稳定在约 2 GB |
| 预防 | 在部署流水线中加入基数检查 | 长期安全 |

监控系统必须比它所监控的系统更可靠。Prometheus 中的高基数标签是一颗静默的内存炸弹——在它引爆之前拆除它。
