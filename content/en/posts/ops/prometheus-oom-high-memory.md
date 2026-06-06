---
title: "Prometheus OOM — When Your Monitoring System Became the Incident"
date: 2026-06-07
weight: 100370
slug: "prometheus-oom-high-memory"
tags: ["prometheus", "monitoring", "troubleshooting", "oom", "performance"]
categories: ["Troubleshooting"]
description: "A Prometheus OOM incident — how a heavily labeled metric and overly broad recording rules caused the Prometheus pod to run out of memory, taking down the entire monitoring stack"
keywords: "prometheus oom, prometheus high memory usage, prometheus tsdb, prometheus label cardinality, prometheus recording rules optimization"
draft: false
featured: true
cover:
  image: ""
  caption: "Prometheus OOM — Troubleshooting"
---

# Prometheus OOM — When Your Monitoring System Became the Incident

## Common Search Queries

| Search Query | Intent |
|---|---|
| prometheus oom | User's Prometheus is crashing due to memory pressure |
| prometheus high memory usage | General troubleshooting for memory leaks or bloat |
| prometheus tsdb memory | Investigating TSDB memory consumption specifically |
| prometheus label cardinality problem | User suspects high-cardinality labels |
| prometheus recording rules optimization | Looking to reduce the cost of recording rules |
| prometheus oom killed kubernetes | Prometheus OOMing in a K8s container |
| prometheus tsdb analyze cardinality | Using promtool to diagnose series explosion |
| prometheus wal replay memory | WAL replay consuming too much memory after restart |
| prometheus heap profile | User wants to profile Prometheus memory |
| prometheus /api/v1/status/tsdb | Checking label series counts via the HTTP API |

---

## The Incident

**Environment:**

| Item | Detail |
|---|---|
| Prometheus version | v2.45 |
| Deployment | Kubernetes (StatefulSet) |
| Memory limit | 8 GB |
| Targets scraped | ~500 |
| Series count | ~1 million |
| Retention | 15 days |
| Scrape interval | 15 s |

It was a quiet Tuesday afternoon. A new microservice — the "order-analytics" service — was deployed into production. Standard procedure: add a ServiceMonitor, let Prometheus discover it, start collecting metrics.

Three hours later, PagerDuty lit up.

**The symptom chain:**

```
15:42  — Grafana dashboards go gray ("No data")
15:43  — Alertmanager fires "Watchdog" alert (heartbeat missing)
15:44  — Watchdog escalates to "AlertmanagerDown"
15:45  — PagerDuty alerts the entire SRE team
```

The Prometheus pod had restarted. Then it restarted again. And again. Every 3-4 hours, like clockwork. Between restarts, there was a brief window where Prometheus would start up, begin scraping, ingest data for a few hours — then die.

The entire monitoring stack was blind. No alerts firing, no dashboards rendering, no metrics available for diagnosis. The monitoring system itself had become the incident.

---

## Background

To understand why Prometheus ran out of memory, you need to understand how it uses memory internally.

### Prometheus Memory Model

| Component | What It Does | Memory Impact |
|---|---|---|
| **TSDB** (Time Series Database) | Stores all time series data on disk with mmap | Moderate — mostly disk-backed |
| **WAL** (Write-Ahead Log) | Temporary buffer for incoming samples before they're compacted into blocks | High during active ingestion |
| **In-memory Chunks** | The most recent samples for each series are held in memory (not yet persisted) | **High** — proportional to series count |
| **Head block** | The "active" TSDB block that accepts writes | High — all series' current chunks live here |
| **Query engine** | Evaluates PromQL queries | Spiky — expensive queries consume memory during evaluation |

The critical insight: **Prometheus keeps one in-memory chunk per actively-scraped series** (typically ~120 samples at 15s scrape interval = 30 minutes of data). If you have 1 million series, that's 1 million chunks in memory. Each chunk is roughly 1-2 KB. The math adds up quickly.

### Cardinality Explained

Cardinality = number of unique label combinations for a metric.

```
http_request_duration_ms{user_id="u1", endpoint="/api/checkout", method="POST", status_code="200", instance="pod-1"}
http_request_duration_ms{user_id="u2", endpoint="/api/checkout", method="POST", status_code="200", instance="pod-1"}
...
```

If a label like `user_id` has 10,000 unique values, and there are 50 endpoints, 5 methods, 10 status codes, and 500 instances — the total series count becomes:

```
10,000 (users) × 50 (endpoints) × 5 (methods) × 10 (status codes) × 500 (instances)
= 12,500,000,000,000 series
```

In practice Prometheus applies deduplication, but even with conservative estimates, adding a high-cardinality label like `user_id` can multiply series count by 1,000x or more.

### Recording Rules

Recording rules pre-compute PromQL expressions and store the results as new time series. They're useful for speeding up dashboards, but they also **consume additional memory** — the result of every recording rule is a new time series that needs its own in-memory chunk, WAL entries, and storage blocks.

---

## Investigation

By the time we stepped in, the Prometheus pod had already restarted 4 times. We had to work fast — the monitoring window was shrinking.

### Step 1: Check Prometheus Pod Logs

```bash
# Check why the pod died
kubectl describe pod prometheus-0 | grep -A5 "Last State"
```

```
Last State: Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Restart Count: 4
```

Exit code 137 = SIGKILL = OOMKilled. The kernel's OOM killer terminated the Prometheus process because the container exceeded its 8 GB memory limit.

```bash
# Check recent logs for clues
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

Scrapes started timing out, then the compaction failed — the process was already being killed.

### Step 2: Check Memory Metrics Before Crash

We needed to see the memory trend. Since Prometheus was down, we relied on kube-state-metrics and node exporter data (stored in a different Prometheus — ironic, monitoring the monitor).

```bash
# Query from a separate monitoring instance
container_memory_working_set_bytes{pod="prometheus-0", namespace="monitoring"}
```

```
12:00 — 3.2 GB
13:00 — 4.8 GB
14:00 — 6.1 GB
15:00 — 7.6 GB
15:42 — OOMKilled (hit 8 GB limit)
```

Steady linear climb. Not a leak — just too many series producing data faster than Prometheus could compact and flush to disk.

### Step 3: Analyze TSDB with promtool

After bringing up a standby Prometheus instance with a reduced scrape config, we ran the cardinality analysis on the TSDB data directory (which had been saved to a persistent volume).

```bash
# Analyze TSDB cardinality
kubectl exec prometheus-0 -- promtool tsdb analyze /prometheus
```

```
Block ID: 01J3XYZ...
  Duration: 2h
  Series: 1,234,567
  ...
  
Label names with highest cardinality:
  user_id:      10,420 unique values  ←  THIS
  instance:        512 unique values
  endpoint:         56 unique values
  status_code:      34 unique values
  method:            8 unique values
```

`user_id` had 10,420 unique values. Combined with other labels, the total series count exceeded 1.2 million.

```bash
# Check total series count per metric
promtool tsdb analyze /prometheus --extended
```

```
Metric with highest number of series:
  http_request_duration_ms:        1,150,000 series  ←  THIS
  http_request_duration_ms_sum:      920,000 series
  http_request_duration_ms_count:    920,000 series
  prometheus_tsdb_head_series:             1  (meta-metric)
  ...
```

The `http_request_duration_ms` metric family alone accounted for 1.15 million series — over 90% of all series in the database.

### Step 4: Check Label Cardinality via API

```bash
# Check label names and their series counts
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

The `memoryInBytesByLabelName` field is telling: just the `user_id` label consumed ~892 MB of memory (label index and postings).

```bash
# Check active series count
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

1,234,567 active series in the head block. With an 8 GB memory limit, that's roughly **6.5 KB per series** — tight, and easily exceeded during WAL replay or query bursts.

### Step 5: Check Recording Rules

```bash
# List recording rules
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

These look innocent at first glance. But `rate(http_request_duration_ms[5m])` aggregates **across all label dimensions** — including `user_id`. This means:

- The recording rule creates a new time series for **every unique label combination** of the source metric
- With 1.15 million source series, the recording rule generates another 1.15 million series
- Double the series count = double the in-memory chunks = double the memory pressure

The issue wasn't just the metric itself — it was the recording rule that expanded (or at least duplicated) the high-cardinality problem.

### Step 6: Check Series Count and Blocks

```bash
# Count blocks and series on disk
kubectl exec prometheus-0 -- promtool tsdb list /prometheus
```

```
BLOCK ULID                  MIN TIME       MAX TIME       DURATION    NUM SAMPLES  NUM CHUNKS  NUM SERIES  SIZE
01J3XYZ...                  2026-06-07-12  2026-06-07-14  2h          850,000,000  1,200,000   1,150,000   4.2 GB
01J3ZAB...                  2026-06-07-10  2026-06-07-12  2h          780,000,000  1,100,000   1,050,000   3.9 GB
...
```

Each 2-hour block was 4+ GB on disk. During compaction, Prometheus loads blocks into memory — with multiple blocks being compacted simultaneously, memory usage can spike by 2-3x.

---

## Root Cause

| Layer | Cause |
|---|---|
| Direct | Container exceeded 8 GB memory limit → OOMKilled |
| Trigger | New `order-analytics` service exposed `http_request_duration_ms` with high-cardinality `user_id` label (10K+ unique values) |
| Series explosion | `http_request_duration_ms` grew from ~50K series to **1.15M series** overnight — a **23x increase** |
| Memory amplification | Recording rules aggregating across all dimensions **doubled** the effective series count in memory |
| Compounding factor | After each OOM restart, WAL replay re-loads all recent samples into memory simultaneously — causing a **memory spike** that hits the limit faster each cycle |
| Why 3-4 hours | That's how long it took Prometheus to ingest enough new series + perform compaction + hit the memory ceiling |

### The Specifics

```
Before deployment:
  Series count: ~50,000
  Memory usage: ~2.5 GB
  Recording rules: 5 rules over ~50K series
  Stability: 30+ days uptime

After deployment:
  New metric: http_request_duration_ms{user_id, request_id, endpoint, method, status_code, instance}
  user_id cardinality: 10K unique values
  New series: 1.15M (500 instances × 50 endpoints × 10 methods × 5 status codes × 10K users — deduplicated)
  Memory usage: 7.6 GB and climbing
  Recording rules: expanded to operate over 1.15M series
  Stability: 3-4 hours before OOM
```

### WAL Replay Death Spiral

A critical compounding effect made things worse:

1. Prometheus OOMs → process exits
2. Pod restarts → Prometheus replays WAL to reconstruct in-memory state
3. WAL replay loads **all recent un-compacted samples** into memory at once
4. Memory spikes to ~6 GB during replay (before even starting normal scrapes)
5. Normal scrapes start, adding even more series
6. Memory climbs past 8 GB → OOM again
7. Repeat, with the WAL growing larger each cycle (because blocks never fully compact)

This is the "death spiral" of Prometheus OOM: **each restart makes the next one come faster** because the WAL keeps growing.

---

## Resolution

### Emergency: Increase Memory Limit (Immediate Relief)

The fastest way to stop the bleeding — buy time for a proper fix.

```bash
kubectl edit statefulset prometheus -n monitoring
```

```yaml
resources:
  limits:
    memory: 16Gi       # Was 8Gi
  requests:
    memory: 12Gi       # Was 6Gi
```

```bash
# Apply and wait for restart
kubectl rollout status statefulset prometheus -n monitoring
```

This immediately stopped the OOM cycling. However, increasing memory without addressing the root cause is just delaying the problem — and adding cost.

### Drop High-Cardinality Labels via Relabel Config

The real fix: drop the `user_id` and `request_id` labels from the problematic metric. These labels are useful for per-request debugging but **should never be stored in a Prometheus TSDB**. They belong in a logging system (Elasticsearch, Loki) or a tracing backend (Jaeger, Tempo), not in a time-series database.

```yaml
# prometheus.yml — relabel_configs
scrape_configs:
  - job_name: "order-analytics"
    ...
    relabel_configs:
      # Drop the high-cardinality label that's killing the TSDB
      - source_labels: [__name__]
        regex: "http_request_duration_ms.*"
        action: labeldrop
        target_label: user_id
      
      # Also drop request_id — same problem
      - source_labels: [__name__]
        regex: "http_request_duration_ms.*"
        action: labeldrop
        target_label: request_id
```

Alternatively, drop at the global level to catch any metric that might carry similar labels:

```yaml
# Global relabel — catch all metrics with problematic labels
relabel_configs:
  - action: labeldrop
    regex: "(user_id|request_id|session_id|transaction_id)"
```

**Important**: `labeldrop` drops the label but keeps the metric. The metric still has `endpoint`, `method`, `status_code`, `instance` — which are more than enough for useful aggregation. You lose per-user granularity, but you gain a working monitoring system.

After applying this config and restarting:

```
Series count: 1.15M → ~25K  (reduced by 98%)
Memory usage: 7.6 GB → ~1.8 GB
```

### Optimize Recording Rules

The recording rules were computing aggregations across all dimensions including high-cardinality labels. After dropping those labels, the rules became much cheaper. But there are additional best practices:

```yaml
# Before (expensive — aggregates across all dimensions including user_id)
groups:
  - name: http_request_recording_rules
    rules:
      - record: "http_request_duration_ms_avg_5m"
        expr: |
          rate(http_request_duration_ms[5m])

# After (cheaper — aggregate by useful dimensions only)
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

Key optimizations:
- **Aggregate early**: `sum by (important_labels)` before recording. Fewer series in the recording rule result = less memory.
- **Limit dimensions**: Only include labels that dashboards actually use.
- **Avoid `by()` without specifying labels**: If you write `rate(http_request_duration_ms[5m])` without an aggregation, each unique label combination gets its own series in the recording rule output — mirroring the source metric's cardinality.

### Adjust TSDB Retention and Block Duration

```bash
# Reduce retention (if 15 days isn't strictly needed)
--storage.tsdb.retention.time=7d

# Control max block duration to limit memory during compaction
--storage.tsdb.max-block-duration=2h

# Minimum block duration (smaller blocks = more compaction but less memory per compaction)
--storage.tsdb.min-block-duration=30m
```

```yaml
# In the StatefulSet args
args:
  - "--storage.tsdb.retention.time=7d"
  - "--storage.tsdb.max-block-duration=2h"
  - "--storage.tsdb.min-block-duration=30m"
```

### Verification

```bash
# 1. Confirm no more OOM kills
kubectl describe pod prometheus-0 | grep -A5 "Last State"
# Should show "Running" with no recent OOM

# 2. Check memory usage
kubectl top pod prometheus-0 -n monitoring
# Should be stable under 4 GB

# 3. Verify series count is under control
kubectl exec prometheus-0 -- wget -qO- 'http://localhost:9090/api/v1/query?query=prometheus_tsdb_head_series'
# Should show dramatically fewer series

# 4. Check that dashboards are working
curl -s http://prometheus:9090/api/v1/query?query=http_request_duration_ms_count | jq '.data.result | length'
# Should return a manageable number of series
```

### Long-Term: Consider Thanos or Cortex

For organizations that genuinely need long-term retention with high cardinality, consider a multi-tier approach:

| Solution | When to Use |
|---|---|
| **Thanos** | Sidecar pattern + object storage; good if you want Prometheus-without-limits |
| **Cortex / Mimir** | Multi-tenant, horizontally scalable; good for large deployments |
| **VictoriaMetrics** | Drop-in Prometheus-compatible replacement with better memory efficiency |
| **Logging for high-cardinality** | `user_id` and `request_id` belong in logs/traces, not metrics |

---

## Lessons Learned

1. **Cardinality is the #1 hidden killer of Prometheus.** A single metric with a high-cardinality label is like a memory leak — gradual, compounding, and catastrophic. Always audit new metrics before deploying them to production. `promtool tsdb analyze` should be part of your CI/CD pipeline.

2. **Never put unbounded labels in Prometheus.** Labels like `user_id`, `request_id`, `session_id`, `email`, `ip_address` — anything with unlimited or end-user-defined values — does not belong in a time-series database. The Prometheus documentation explicitly warns about this, and ignoring it leads directly to OOM.

   > ⚠️ **Prometheus documentation warning**: "Be mindful of the cardinality of your labels. A label with many different values will cause Prometheus to use a lot of memory."

3. **Recording rules amplify cardinality problems.** If your source metric has high cardinality, a recording rule that aggregates across all dimensions will create equally high cardinality in its output. Always `sum by (important_labels)` to reduce dimensionality before recording.

4. **Monitoring the monitor is not optional.** When Prometheus goes down, you go blind. Set up a separate, lightweight Prometheus (or use a managed service) to monitor the main Prometheus — memory usage, series count, scrape failures. Without it, you're debugging an OOM with zero metrics, which is almost impossible.

5. **WAL replay is the death spiral.** If Prometheus OOMs once, it will OOM faster the next time because the WAL grows during each incomplete cycle. Breaking this cycle requires either (a) drastically reducing the scrape load before restarting, or (b) increasing memory to survive the replay.

6. **8 GB was too tight for 1M series.** A rough rule of thumb: each million series consumes approximately 4-6 GB of memory just for the head block, plus overhead for WAL, queries, and compaction. Plan for at least 8-10 GB per million series — and that's before recording rules.

---

## Summary

| Phase | Key Action | Result |
|---|---|---|
| Detection | OOMKilled pod, PagerDuty escalation | Incident declared |
| Triage | `kubectl describe pod` → OOMKilled | Root cause direction found |
| Diagnosis | `promtool tsdb analyze` → 1.15M series from `http_request_duration_ms` with `user_id` label | Root cause identified |
| Emergency fix | Increase memory from 8 Gi to 16 Gi | OOM stopped immediately |
| Root fix | `labeldrop user_id` and `request_id` via relabel_config | Series count reduced by 98% |
| Optimization | Rewrite recording rules with explicit `by()` aggregations | Memory usage stable at ~2 GB |
| Prevention | Add cardinality checks to deployment pipeline | Long-term safety |

The monitoring system must be more reliable than the systems it monitors. High-cardinality labels in Prometheus are a silent memory bomb — defuse them before they detonate.
