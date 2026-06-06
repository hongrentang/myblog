---
title: "Elasticsearch 集群状态 RED 排查实录 — 磁盘水位配置不当导致半数分片丢失、搜索服务全面瘫痪"
date: 2026-06-06
weight: 100350
slug: "elasticsearch-cluster-red-status"
tags: ["elasticsearch", "database", "troubleshooting", "cluster", "search"]
categories: ["Troubleshooting"]
description: "一次 Elasticsearch 集群 RED 状态故障处理实录——磁盘水位线配置失误导致全部副本分片无法分配，生产搜索服务全面宕机"
keywords: "elasticsearch 集群 red, es 分片分配, elasticsearch 磁盘水位, es 未分配分片, elasticsearch 故障排查"
draft: false
featured: true
cover:
  image: ""
  caption: "Elasticsearch 集群 RED — 故障排查"
---

---

## 常见搜索查询 / Common Search Queries

- elasticsearch 集群状态 red 如何修复
- es 分片分配失败 原因
- elasticsearch disk watermark 配置详解
- elasticsearch 未分配分片 解决方法
- es 集群 red 磁盘满了怎么办
- elasticsearch 副本分片 无法分配
- elk 集群健康 red 修复
- ES 节点磁盘使用率过高 分片未分配

---

## 故障经过 / The Incident

### 环境信息

| 项目       | 详情                              |
|-----------|-----------------------------------|
| Elasticsearch | 7.17（生产集群）               |
| 节点数    | 3                                 |
| 索引数    | 10                                |
| 主分片数  | 50                                |
| 副本分片数 | 50                               |
| 总分片数  | 100                               |
| 业务角色  | 生产 SaaS 平台全文搜索引擎           |

### 时间线

**14:23 (UTC+8)** — 值班工程师收到 P0 级告警：产品搜索 API 开始返回 HTTP 500 错误。用户反馈 Web 应用中的搜索结果不完整甚至完全空白。

**14:25** — 检查 Kibana 监控面板。集群概览页面顶部显示红色大横幅：**Cluster Health: RED**。多个索引被标记为红色状态。

**14:27** — 初步判断：某个节点可能已宕机，或磁盘空间已耗尽。

### 故障表现

- 产品搜索 API 向客户端返回 **HTTP 500** 错误
- API 完全中断前，用户看到了不完整的搜索结果
- Kibana 显示 **Cluster Health: RED**
- 索引管理界面中部分索引显示为 `red`
- 50 个副本分片全部处于 `UNASSIGNED` 状态
- 告警系统检测到搜索服务错误率飙升

---

## 背景 / Background

### Elasticsearch 集群架构

Elasticsearch 是一个分布式系统，数据按 **索引 (Index)** 组织。每个索引拆分为多个 **分片 (Shard)**，每个分片可以有一个或多个 **副本 (Replica)** 以实现高可用。

- **主分片 (Primary shard)**：数据子集的权威副本，读写操作经过主分片。
- **副本分片 (Replica shard)**：主分片的副本，提供故障转移和读取扩展能力。
- **节点 (Node)**：单个 Elasticsearch 实例，集群需要多节点实现高可用。

本例中：10 个索引，每个 5 个主分片（共 50 个主分片），各 1 个副本（共 50 个副本分片）。3 个节点共同承载 100 个分片。

### 分片分配机制

当节点加入或离开集群时，Elasticsearch 的 **分配决策器 (Allocation Decider)** 决定分片的放置位置。分配过程考虑：

1. **磁盘可用空间** — 节点必须有足够剩余磁盘
2. **分片过滤** — 基于节点属性的路由规则
3. **再均衡** — 分片的均匀分布
4. **并发恢复** — 并行分片恢复的数量限制

### 磁盘水位线设置

Elasticsearch 定义了三个关键磁盘水位线阈值，基于磁盘使用率控制分片分配：

| 设置项                                              | 默认值  | 说明                                             |
|-----------------------------------------------------|--------|--------------------------------------------------|
| `cluster.routing.allocation.disk.watermark.low`     | 85%    | 磁盘使用超过此值后，不再向该节点分配新分片。        |
| `cluster.routing.allocation.disk.watermark.high`    | 90%    | 磁盘使用超过此值后，开始将分片迁移到其他节点。      |
| `cluster.routing.allocation.disk.watermark.flood_stage` | 95% | 强制将索引设为只读。该节点上的所有分片变为只读。     |

当达到 flood_stage 水位线时，Elasticsearch 强制将所有在该节点上有分片的索引设为只读 (`index.blocks.read_only_allow_delete`)。这是防止磁盘完全耗尽的最底线保护机制。

---

## 排查过程 / Investigation

### 第一步：检查集群健康状态

```json
GET _cluster/health
```

返回结果：

```json
{
  "cluster_name": "production-search",
  "status": "red",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 30,
  "active_shards": 30,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 70,
  "delayed_unassigned_shards": 0,
  "number_of_pending_tasks": 7,
  "number_of_in_flight_fetch": 0,
  "task_max_waiting_in_queue_millis": 4523,
  "active_shards_percent_as_number": 30.0
}
```

关键发现：
- **状态: red** — 至少有一个主分片未分配
- **活跃主分片: 30** — 50 个主分片中只有 30 个可用
- **未分配分片: 70** — 大量分片（主分片 + 副本分片）未分配
- **活跃分片: 30** — 100 个分片中仅 30 个活跃

### 第二步：查看未分配分片的原因

```json
GET _cluster/allocation/explain
```

返回结果：

```json
{
  "index": "products_v2",
  "shard": 0,
  "primary": true,
  "current_state": "unassigned",
  "unassigned_info": {
    "reason": "NODE_LEFT",
    "at": "2026-06-06T06:23:34.712Z",
    "last_allocation_status": "no_attempt"
  },
  "can_allocate": "no",
  "allocate_explanation": "cannot allocate because allocation is not permitted to any of the nodes",
  "node_allocation_decisions": [
    {
      "node_id": "k7xJ3YbSRTy0zP9cV2n1Fg",
      "node_name": "es-data-01",
      "store_enough": false,
      "store_explanation": "the node is above the low watermark cluster setting [85%], using more disk space than the maximum allowed [95.0%] (actual free: 4.0%)",
      "deciders": [
        {
          "decider": "disk_threshold",
          "decision": "NO",
          "explanation": "the node is above the low watermark cluster setting [85%], using more disk space than the maximum allowed [95.0%]"
        }
      ]
    },
    {
      "node_id": "L3pQ6aRkTzY9xW2bN5mHg",
      "node_name": "es-data-02",
      "store_enough": false,
      "store_explanation": "the node is above the low watermark cluster setting [85%], using more disk space than the maximum allowed [92.0%] (actual free: 8.0%)",
      "deciders": [
        {
          "decider": "disk_threshold",
          "decision": "NO",
          "explanation": "the node is above the low watermark cluster setting [85%], using more disk space than the maximum allowed [92.0%]"
        }
      ]
    },
    {
      "node_id": "R8tF2dKpVnL5cH7sM3xBj",
      "node_name": "es-data-03",
      "store_enough": true,
      "store_explanation": "the node is below the low watermark cluster setting [85%] (actual free: 25.0%)",
      "deciders": [
        {
          "decider": "disk_threshold",
          "decision": "YES"
        }
      ]
    }
  ]
}
```

这揭示了核心问题：**只有 es-data-03 一个节点有足够的剩余磁盘接受新分片**，但单个节点无法同时承载同一个分片的主分片和副本分片。

### 第三步：检查所有节点的磁盘使用率

```json
GET _cat/nodes?v=true&h=name,disk.total,disk.used,disk.avail,disk.used_percent,disk.percent
```

返回结果：

| name        | disk.total | disk.used | disk.avail | disk.used_percent | disk.percent |
|-------------|------------|-----------|------------|-------------------|--------------|
| es-data-01  | 500.0gb    | 480.0gb   | 20.0gb     | 96.0              | 96           |
| es-data-02  | 500.0gb    | 460.0gb   | 40.0gb     | 92.0              | 92           |
| es-data-03  | 500.0gb    | 375.0gb   | 125.0gb    | 75.0              | 75           |

**es-data-01**: 使用率 96% — 超出 low (85%)、high (90%) 和 flood_stage (95%) 全部水位线。
**es-data-02**: 使用率 92% — 超出 low (85%) 和 high (90%) 水位线。

### 第四步：检查水位线配置

```json
GET _cluster/settings?include_defaults=true&flat_settings=true
```

相关返回结果：

```json
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "95%",
    "cluster.routing.allocation.disk.watermark.high": "97%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
  }
}
```

这是关键证据。水位线被人为修改到了危险的高位：

- **low: 95%**（默认 85%）— 分片会持续分配到节点，直到磁盘使用率达到 95%
- **high: 97%**（默认 90%）— 磁盘使用率达到 97% 才开始迁移分片
- **flood_stage: 98%**（默认 95%）— 磁盘使用率达到 98% 才强制索引只读

这些值完全破坏了磁盘水位线的保护作用。

### 第五步：通过节点 API 验证磁盘空间

```bash
curl -s "http://localhost:9200/_nodes/stats/fs?pretty" | jq '.nodes[] | {name: .name, "total_in_bytes": .fs.total.total_in_bytes, "available_in_bytes": .fs.total.available_in_bytes, "used_percent": ((.fs.total.total_in_bytes - .fs.total.available_in_bytes) / .fs.total.total_in_bytes * 100 | floor) }'
```

### 第六步：检查 Elasticsearch 日志

在 es-data-01 上，日志反复出现水位线警告：

```
[2026-06-06T06:23:34,712][WARN ][o.e.c.r.a.d.DiskThresholdMonitor] [es-data-01] high disk watermark [90%] exceeded on node [es-data-01] free: 4.9gb (4.0%), replicas will not be assigned to this node
[2026-06-06T06:23:35,113][WARN ][o.e.c.r.a.d.DiskThresholdMonitor] [es-data-01] flood stage disk watermark [95%] exceeded on node [es-data-01] free: 4.9gb (4.0%), all indices on this node will be marked read-only
[2026-06-06T06:23:36,419][WARN ][o.e.c.r.a.DiskThresholdDecider] [es-data-01] after shard started, node exceeded high watermark, shard will be relocated: shard=[products_v2][0] node=[es-data-01]
```

---

## 根因分析 / Root Cause

根本原因是一条 **配置错误引发的级联故障链**：

1. **磁盘水位线配置错误** — 集群的 `cluster.routing.allocation.disk.watermark.low` 被设为 95%（默认 85%），`high` 设为 97%（默认 90%）。此前有人为了提高磁盘利用率调整了这些值，但修改后从未针对集群的数据增长速度进行验证。

2. **es-data-01 磁盘耗尽** — 一次常规数据写入导致 es-data-01 磁盘使用率达到 96%。由于 low 水位线设在 95%，该节点无法再接受新副本分片。但 high 水位线设在 97%，Elasticsearch 也没有主动将分片迁离。

3. **节点暂时离开集群** — es-data-01 上的 Elasticsearch 进程因极端磁盘压力（inode 几乎耗尽、写入能力接近零）变得不稳定，节点暂时从集群断开。

4. **es-data-01 上所有分片变为未分配** — 节点下线后，该节点承载的 20 个主分片和 20 个副本分片全部变为 `UNASSIGNED`。

5. **集群状态 RED** — 原本在 es-data-01 上的部分主分片无法分配到剩余节点（es-data-02 使用率 92% 已超过 low 水位线，无法接受新分片；es-data-03 有空间但无法同时承载同一分片的主副本两个副本），集群进入 RED 状态。

6. **连锁反应波及剩余节点** — es-data-02 使用率已达 92%，同样超过了配置错误的 low 水位线，分配决策器阻止了任何新分片分配。仅 es-data-03（使用率 75%）能接受分片，但单节点无法满足所有分片的副本分配要求。

---

## 解决方案 / Resolution

### 第一步：紧急释放磁盘空间 + 重试分配

```bash
# 清理未被轮转的 Elasticsearch 日志
sudo journalctl --vacuum-time=1d

# 从节点的本地缓存中删除旧的 Elasticsearch 快照
curl -X DELETE "localhost:9200/_snapshot/my_repository/old_snapshot_$(date -d '-7 days' +%Y%m%d)"

# 强制合并旧索引以回收空间（仅对非关键的只读索引执行）
curl -X POST "localhost:9200/logs-2025-*/_forcemerge?max_num_segments=1"
```

然后重试失败的分配：

```json
POST _cluster/reroute?retry_failed
```

### 第二步：修复水位线配置

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}
```

或者使用字节值进行更精细的控制：

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "75gb",
    "cluster.routing.allocation.disk.watermark.high": "50gb",
    "cluster.routing.allocation.disk.watermark.flood_stage": "25gb",
    "cluster.routing.allocation.disk.watermark.flood_stage.frozen": "25gb",
    "cluster.routing.allocation.disk.watermark.enable_for_single_data_node": true
  }
}
```

### 第三步：重新启用分配并重试

```json
POST _cluster/reroute?retry_failed
```

### 第四步：监控恢复过程

```json
GET _cluster/health?wait_for_status=green&timeout=120s
```

```json
GET _cat/recovery?active_only=true&h=index,shard,type,stage,bytes_percent
```

大约 4 分钟后，集群恢复到 **GREEN** 状态：

```json
{
  "cluster_name": "production-search",
  "status": "green",
  "timed_out": false,
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 50,
  "active_shards": 100,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "unassigned_shards": 0,
  "active_shards_percent_as_number": 100.0
}
```

### 第五步：长期预防措施

1. **磁盘监控告警**：
   - 搭建 Prometheus + Grafana，监控所有 ES 节点的磁盘使用率
   - 磁盘使用率达到 75% 告警（警告），85% 告警（严重）
   - 监控 `elasticsearch_cluster_health_status` 指标，值不为 green 时告警

2. **使用 Curator/ILM 进行索引生命周期管理**：
   ```yaml
   # curator action file: delete_old_indices.yml
   actions:
     1:
       action: delete_indices
       description: "删除 30 天前的旧索引"
       options:
         ignore_empty_list: true
         disable_action: false
       filters:
       - filtertype: age
         source: creation_date
         direction: older
         unit: days
         unit_count: 30
       - filtertype: pattern
         kind: regex
         value: '^(logs-|metrics-)'
   ```

3. **定期配置审计**：
   - 每季度审查集群设置
   - 使用 `elasticsearch-config-check.sh` 或类似的验证工具
   - 记录所有非默认配置及其修改理由

4. **跨节点磁盘均衡**：
   - 确保分片在各节点间平衡分布
   - 必要时使用 `_cluster/reroute` 的 `move` 命令手动再均衡
   - 监控各节点磁盘使用率，当任意两个数据节点差异超过 15% 时告警

---

## 经验教训 / Lessons Learned

1. **未经充分测试，切勿覆盖默认磁盘水位线。** 默认值的存在自有其道理。如果必须调整，先在模拟生产的预发布环境中进行基准测试，并至少监控一个完整的数据保留周期。

2. **磁盘监控不可妥协。** 集群在 es-data-02 使用率已达 92% 的情况下运行了数周。一个简单的按节点展示磁盘使用率的 Grafana 仪表盘，就能及早发现这一趋势。

3. **分片分配决策是节点级的，而非集群级的。** 分配决策器独立评估每个节点。虽然集群总空闲空间足够，但分布严重不均，大多数节点已超过分配阈值。

4. **记录所有配置变更。** 没有人记得是谁、在什么时候修改了水位线设置。所有配置变更应纳入版本控制并经过审核。

5. **测试恢复流程。** 修复水位线后，恢复过程暴露出部分索引因强制停机存在损坏的段。定期的快照恢复演练可以验证备份策略是否有效。

6. **单个配置错误可以击穿整个安全体系。** 三层水位线（low、high、flood_stage）正是为此类场景设计。将三者全部设为不安全水平，相当于彻底禁用了所有保护机制。

---

## 总结 / Summary

```
                    ┌──────────────────────────────┐
                    │       水位线配置错误          │
                    │  low=95%  high=97%  flood=98% │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  es-data-01 磁盘使用率达 96%  │
                    │  （超出 low 水位线）           │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  节点不稳定，从集群断开         │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  40 个分片变为 UNASSIGNED     │
                    │  （20 主分片 + 20 副本分片）   │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  无法重新分配：               │
                    │  es-data-01: 96%（已满）     │
                    │  es-data-02: 92%（超低水位） │
                    │  es-data-03: 唯一空节点       │
                    │  （不能同时放主副分片）        │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │     集群状态：RED             │
                    │   搜索 API → HTTP 500        │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────┴───────────────┐
                    │         解决方案              │
                    │  1. 释放 es-data-01 磁盘空间  │
                    │  2. 重置水位线：              │
                    │     low=85% high=90% flood=95%│
                    │  3. POST _cluster/reroute     │
                    │  4. 等待 GREEN                │
                    │  5. 添加监控告警              │
                    └──────────────────────────────┘
```

---

### 快速参考命令

```bash
# 1. 检查集群健康状态
curl -s "localhost:9200/_cluster/health" | jq .

# 2. 查看分片未分配原因
curl -s "localhost:9200/_cluster/allocation/explain" | jq .

# 3. 检查各节点磁盘使用率
curl -s "localhost:9200/_cat/nodes?v=true&h=name,disk.total,disk.used,disk.avail,disk.used_percent,disk.percent"

# 4. 检查当前水位线设置
curl -s "localhost:9200/_cluster/settings?include_defaults=true&flat_settings=true" | jq '.[] | with_entries(select(.key | contains("watermark")))'

# 5. 修复水位线（持久化）
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}'

# 6. 重试未分配分片的分配
curl -X POST "localhost:9200/_cluster/reroute?retry_failed"

# 7. 等待集群恢复 GREEN
curl -s "localhost:9200/_cluster/health?wait_for_status=green&timeout=120s" | jq .

# 8. 查看恢复进度
curl -s "localhost:9200/_cat/recovery?active_only=true&h=index,shard,type,stage,bytes_percent"
```
