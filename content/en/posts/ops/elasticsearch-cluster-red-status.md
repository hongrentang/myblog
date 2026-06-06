---
title: "Elasticsearch Cluster RED — When Half Your Shards Went Missing and Search Broke"
date: 2026-06-06
weight: 100350
slug: "elasticsearch-cluster-red-status"
tags: ["elasticsearch", "database", "troubleshooting", "cluster", "search"]
categories: ["Troubleshooting"]
description: "An Elasticsearch cluster RED status incident — how a disk watermark misconfiguration caused all replica shards to be unassigned, taking down the production search functionality"
keywords: "elasticsearch cluster red, es shard allocation, elasticsearch disk watermark, es unassigned shards, elasticsearch troubleshooting"
draft: false
featured: true
cover:
  image: ""
  caption: "Elasticsearch Cluster RED — Troubleshooting"
---

---

## Common Search Queries

- elasticsearch cluster red status fix
- es shard allocation failed
- elasticsearch disk watermark low high flood_stage
- elasticsearch unassigned shards troubleshooting
- es cluster red after node disk full
- elasticsearch replica shards not assigned
- why is my elasticsearch cluster red
- elk cluster health red fix

---

## The Incident

### Environment

| Item               | Detail                                  |
|--------------------|-----------------------------------------|
| Elasticsearch      | 7.17 (production cluster)               |
| Nodes              | 3                                       |
| Indices            | 10                                      |
| Primary Shards     | 50                                      |
| Replica Shards     | 50                                      |
| Total Shards       | 100                                     |
| Role               | Full-text search for a production SaaS  |

### Timeline

**14:23 (UTC+8)** — The on-call engineer received a P0 alert: the product search API began returning HTTP 500 errors. Users reported that search results were incomplete or missing entirely in the web application.

**14:25** — Kibana dashboards were checked. The cluster overview showed a large red banner across the top: **Cluster Health: RED**. Several indices were marked with red status.

**14:27** — Initial suspicion: a node may have gone down or disk space may have been exhausted.

### Symptoms

- Product search API returning **HTTP 500** errors to clients
- Incomplete search results returned to users before the API fully broke
- Kibana **Cluster Health: RED** indicator
- Some indices appeared as `red` in the index management UI
- 50 replica shards showing `UNASSIGNED` status
- Alerting system detected elevated error rates on the search service

---

## Background

### Elasticsearch Cluster Architecture

An Elasticsearch cluster is a distributed system where data is organized into **indices**. Each index is split into **shards** (the unit of distribution), and each shard can have one or more **replicas** for high availability.

- **Primary shard**: The authoritative copy of a data subset. Reads and writes go through the primary.
- **Replica shard**: A copy of the primary shard. Provides failover and read scalability.
- **Node**: A single running instance of Elasticsearch. A cluster must have multiple nodes for high availability.

In our setup: 10 indices, 5 primary shards each (total 50 primaries), 1 replica each (total 50 replicas). With 3 nodes, Elasticsearch distributes the 100 shards across the nodes.

### Shard Allocation

When a node joins or leaves the cluster, Elasticsearch's **allocation decider** decides where to place shards. The allocation process considers:

1. **Disk availability** — nodes must have enough free disk
2. **Shard filtering** — routing rules based on node attributes
3. **Rebalancing** — even distribution of shards
4. **Concurrent recoveries** — limits on parallel shard recovery

### Disk Watermark Settings

Elasticsearch has three critical disk watermark thresholds that control shard allocation based on disk usage:

| Setting                                            | Default | Description                                                       |
|----------------------------------------------------|---------|-------------------------------------------------------------------|
| `cluster.routing.allocation.disk.watermark.low`    | 85%     | Stop allocating shards to a node when disk usage exceeds this.    |
| `cluster.routing.allocation.disk.watermark.high`   | 90%     | Start moving shards off a node when disk usage exceeds this.      |
| `cluster.routing.allocation.disk.watermark.flood_stage` | 95% | Force indices to read-only. All shards on the node become read-only. |

When the flood stage is crossed, Elasticsearch forces every index with shards on that node into read-only mode (`index.blocks.read_only_allow_delete`). This is a last-resort protection to prevent complete disk exhaustion.

---

## Investigation

### Step 1: Check Cluster Health

```json
GET _cluster/health
```

Response:

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

Key observations:
- **Status: red** — one or more primary shards are unassigned
- **Active primary shards: 30** — only 30 out of 50 primaries are active
- **Unassigned shards: 70** — a massive number of unassigned shards (primaries + replicas)
- **Active shards: 30** — only 30 out of 100 shards are active

### Step 2: Explain Unassigned Shards

```json
GET _cluster/allocation/explain
```

Response:

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

This revealed the critical problem: **only one node (es-data-03) had enough free disk to accept new shards**, but a single node cannot host both a primary and its replica (same shard cannot be allocated to the same node).

### Step 3: Check Disk Usage on All Nodes

```json
GET _cat/nodes?v=true&h=name,disk.total,disk.used,disk.avail,disk.used_percent,disk.percent
```

Response:

| name        | disk.total | disk.used | disk.avail | disk.used_percent | disk.percent |
|-------------|------------|-----------|------------|-------------------|--------------|
| es-data-01  | 500.0gb    | 480.0gb   | 20.0gb     | 96.0              | 96           |
| es-data-02  | 500.0gb    | 460.0gb   | 40.0gb     | 92.0              | 92           |
| es-data-03  | 500.0gb    | 375.0gb   | 125.0gb    | 75.0              | 75           |

**es-data-01**: 96% used — past both low watermark (85%) and high watermark (90%), and at flood_stage (95%).
**es-data-02**: 92% used — past low watermark (85%) and high watermark (90%).

### Step 4: Check Watermark Configuration

```json
GET _cluster/settings?include_defaults=true&flat_settings=true
```

Relevant response:

```json
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "95%",
    "cluster.routing.allocation.disk.watermark.high": "97%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "98%"
  }
}
```

This was the smoking gun. The watermarks had been manually set to dangerously high levels:

- **low: 95%** (default is 85%) — shards would keep being allocated to a node until disk was 95% full
- **high: 97%** (default is 90%) — shards would only start being moved off when disk reached 97%
- **flood_stage: 98%** (default is 95%) — indices would not be forced read-only until 98%

These values defeated the purpose of disk watermarks entirely.

### Step 5: Check Node-Level Disk via Node API

```bash
curl -s "http://localhost:9200/_nodes/stats/fs?pretty" | jq '.nodes[] | {name: .name, "total_in_bytes": .fs.total.total_in_bytes, "available_in_bytes": .fs.total.available_in_bytes, "used_percent": ((.fs.total.total_in_bytes - .fs.total.available_in_bytes) / .fs.total.total_in_bytes * 100 | floor) }'
```

### Step 6: Check Elasticsearch Logs

On es-data-01, the logs showed repeated watermark warnings:

```
[2026-06-06T06:23:34,712][WARN ][o.e.c.r.a.d.DiskThresholdMonitor] [es-data-01] high disk watermark [90%] exceeded on node [es-data-01] free: 4.9gb (4.0%), replicas will not be assigned to this node
[2026-06-06T06:23:35,113][WARN ][o.e.c.r.a.d.DiskThresholdMonitor] [es-data-01] flood stage disk watermark [95%] exceeded on node [es-data-01] free: 4.9gb (4.0%), all indices on this node will be marked read-only
[2026-06-06T06:23:36,419][WARN ][o.e.c.r.a.DiskThresholdDecider] [es-data-01] after shard started, node exceeded high watermark, shard will be relocated: shard=[products_v2][0] node=[es-data-01]
```

---

## Root Cause

The root cause chain can be summarized as a **configuration error leading to cascading failure**:

1. **Misconfigured disk watermarks** — The cluster had `cluster.routing.allocation.disk.watermark.low` set to 95% (default: 85%) and `high` set to 97% (default: 90%). These values were previously adjusted in an attempt to maximize disk utilization, but the change was never validated against the cluster's data growth rate.

2. **Disk exhaustion on es-data-01** — A routine data ingestion cycle pushed disk usage on es-data-01 to 96%. Because the low watermark was at 95%, no replicas could be allocated to this node. However, the high watermark at 97% meant Elasticsearch did not proactively relocate shards away.

3. **Node temporarily left cluster** — The Elasticsearch process on es-data-01 became unstable due to the extreme disk pressure (near 100% inodes and near-zero write capacity). The node temporarily disconnected from the cluster.

4. **All shards on es-data-01 became unassigned** — When the node dropped, all 20 primary shards and 20 replica shards hosted on es-data-01 became unassigned.

5. **Cluster status RED** — Because some primary shards previously on es-data-01 could not be reassigned to the remaining nodes (es-data-02 was at 92% — past the low watermark — and could not accept new shards; es-data-03 had space but could not host both primary and replica copies of the same shard), the cluster went RED.

6. **Cascade to remaining nodes** — es-data-02, already at 92% usage, was also past the misconfigured low watermark, so the allocation decider blocked any new shard assignments to it. Only es-data-03 (75% used) could accept shards, but a single node cannot satisfy replica allocation for every shard.

---

## Resolution

### Step 1: Emergency — Free Disk Space and Retry Allocation

```bash
# Clean up Elasticsearch logs that were not rotated
sudo journalctl --vacuum-time=1d

# Remove old Elasticsearch snapshots from the node's local cache
curl -X DELETE "localhost:9200/_snapshot/my_repository/old_snapshot_$(date -d '-7 days' +%Y%m%d)"

# Force merge old indices to reclaim space (only do this on non-critical, read-only indices)
curl -X POST "localhost:9200/logs-2025-*/_forcemerge?max_num_segments=1"
```

Then retry failed allocations:

```json
POST _cluster/reroute?retry_failed
```

### Step 2: Fix Watermark Configuration

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

Or set them in bytes for more precise control:

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

### Step 3: Re-enable Allocation and Retry

```json
POST _cluster/reroute?retry_failed
```

### Step 4: Monitor Recovery

```json
GET _cluster/health?wait_for_status=green&timeout=120s
```

```json
GET _cat/recovery?active_only=true&h=index,shard,type,stage,bytes_percent
```

After approximately 4 minutes, the cluster returned to **GREEN** status:

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

### Step 5: Long-term Preventive Measures

1. **Disk monitoring alerts**:
   - Set up Prometheus + Grafana to monitor disk usage on all ES nodes
   - Alert at 75% (warning) and 85% (critical) disk usage
   - Alert on `elasticsearch_cluster_health_status` metric with value != green

2. **Index lifecycle management with Curator/ILM**:
   ```yaml
   # curator action file: delete_old_indices.yml
   actions:
     1:
       action: delete_indices
       description: "Delete indices older than 30 days"
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

3. **Regular configuration audit**:
   - Review cluster settings quarterly
   - Use `elasticsearch-config-check.sh` or a comparable validation tool
   - Document all non-default settings with justification

4. **Cross-node disk balancing**:
   - Ensure shard distribution is balanced across nodes
   - Use `_cluster/reroute` with `move` commands for manual rebalancing when needed
   - Monitor disk usage per node with alerts for significant skew (difference > 15% between any two data nodes)

---

## Lessons Learned

1. **Never override default disk watermarks without thorough testing.** The defaults exist for a reason. If you must change them, benchmark the change in a staging environment that mirrors production, and monitor for at least one full data retention cycle.

2. **Disk monitoring is non-negotiable.** The cluster was operating for weeks with es-data-02 already at 92%. A simple Grafana dashboard with disk usage per node would have caught this trend early.

3. **Shard allocation decisions are node-level, not cluster-level.** The allocation decider evaluates each node independently. Even though the cluster had enough total free space, the distribution was so skewed that most nodes were above the allocation threshold.

4. **Document configuration changes.** Nobody remembered who changed the watermark settings or when. All configuration changes should be tracked in version control and reviewed.

5. **Test the recovery.** After fixing the watermarks, the recovery process revealed that some indices had corrupted segments from the forced shutdown. Regular snapshot/restore drills would have validated that our backup strategy worked.

6. **A single misconfiguration can defeat an entire safety system.** Three layers of watermarks (low, high, flood_stage) are designed to prevent exactly this scenario. By setting all three to unsafe levels, we effectively disabled all of them.

---

## Summary

```
                    ┌──────────────────────────────┐
                    │  Misconfigured Watermarks    │
                    │  low=95%  high=97%  flood=98%│
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  es-data-01 disk reaches 96% │
                    │  (past low watermark)        │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  Node becomes unstable,      │
                    │  disconnects from cluster    │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  40 shards become UNASSIGNED │
                    │  (20 primaries + 20 replicas)│
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │  Cannot reassign:            │
                    │  es-data-01: 96% (full)     │
                    │  es-data-02: 92% (past low) │
                    │  es-data-03: only node free  │
                    │  (can't host prim+replica)   │
                    └──────────────┬───────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │     CLUSTER STATUS: RED      │
                    │   Search API → HTTP 500      │
                    └──────────────┬───────────────┘
                                   │
                    ┌──────────────┴───────────────┐
                    │        RESOLUTION            │
                    │  1. Free disk on es-data-01  │
                    │  2. Reset watermarks:        │
                    │     low=85% high=90% flood=95%│
                    │  3. POST _cluster/reroute    │
                    │  4. Wait for GREEN           │
                    │  5. Add monitoring alerts    │
                    └──────────────────────────────┘
```

---

### Quick Reference Card

```bash
# 1. Check cluster health
curl -s "localhost:9200/_cluster/health" | jq .

# 2. Explain why shards are unassigned
curl -s "localhost:9200/_cluster/allocation/explain" | jq .

# 3. Check disk usage per node
curl -s "localhost:9200/_cat/nodes?v=true&h=name,disk.total,disk.used,disk.avail,disk.used_percent,disk.percent"

# 4. Check current watermark settings
curl -s "localhost:9200/_cluster/settings?include_defaults=true&flat_settings=true" | jq '.[] | with_entries(select(.key | contains("watermark")))'

# 5. Fix watermarks (persistent)
curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
{
  "persistent": {
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}'

# 6. Retry unassigned shard allocation
curl -X POST "localhost:9200/_cluster/reroute?retry_failed"

# 7. Wait for green
curl -s "localhost:9200/_cluster/health?wait_for_status=green&timeout=120s" | jq .

# 8. Check recovery progress
curl -s "localhost:9200/_cat/recovery?active_only=true&h=index,shard,type,stage,bytes_percent"
```
