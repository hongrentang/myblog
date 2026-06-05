---
title: "Kafka Consumer Group Rebalance — How Frequent Rebalances Broke Our Real-Time Pipeline"
date: 2026-06-05
weight: 100340
slug: "kafka-consumer-group-rebalance"
tags: ["kafka", "middleware", "troubleshooting", "consumer", "rebalance"]
categories: ["Troubleshooting"]
description: "A Kafka consumer group rebalance incident — how frequent rebalances caused by improper session.timeout.ms and max.poll.interval.ms settings brought the real-time data pipeline to a halt"
keywords: "kafka consumer group rebalance, kafka rebalance troubleshooting, kafka consumer timeout, kafka session.timeout.ms, kafka max.poll.interval.ms"
draft: false
featured: true
cover:
  image: ""
  caption: "Kafka Consumer Group Rebalance — Troubleshooting"
---

# Kafka Consumer Group Rebalance — How Frequent Rebalances Broke Our Real-Time Pipeline

## Common Search Queries

If you arrived here searching for any of the following, you are in the right place:

- "kafka consumer group rebalance keeps happening"
- "kafka rebalance timeout too frequent"
- "kafka consumer lag spike rebalance"
- "kafka session.timeout.ms too low rebalance"
- "kafka max.poll.interval.ms exceeded rebalance"
- "kafka stop the world rebalance"
- "kafka consumer group not stable"
- "kafka rebalance rate high"
- "kafka consumer keeps leaving and joining group"

## The Incident

**Environment**: Kafka 3.4, 12 consumers in a single consumer group, 24 partitions, real-time analytics pipeline processing user click events. Consumer application deployed on Kubernetes with 4 vCPUs and 8 GB RAM per pod, running Java 17 with Avro deserialization and enrichment logic.

**Time**: Peak hours, 14:00 on a Tuesday. Traffic was at 2x normal load due to a promotional campaign.

**Symptoms**:

The monitoring dashboard lit up within minutes:

| Metric | Before (Normal) | After (Incident) |
|--------|-----------------|------------------|
| Consumer Lag | ~100 messages | **500,000+ messages** |
| Processing Latency (p99) | 2 seconds | **15 minutes** |
| Rebalance Rate | ~0/hr (stable) | **30+/hr** |
| Consumer Group Status | Stable | Continuously rebalancing |

```bash
# What we saw in the monitoring console
# Consumer lag chart went from flatline to hockey-stick growth
```

The data pipeline was supposed to deliver user click events for real-time analytics within seconds. Instead, the team started getting alerts that the analytics dashboard was showing data from 15 minutes ago. The downstream real-time machine learning model was ingesting stale features, impacting recommendation quality.

**Impact**:
- Real-time analytics dashboard delayed by 15+ minutes
- ML model serving degraded due to stale training features
- Downstream billing pipeline experienced data skew
- On-call team paged at 14:05

## Background

### Kafka Consumer Group Architecture

A Kafka consumer group is a set of consumers that collaboratively consume messages from a set of topic partitions. Each partition is assigned to exactly one consumer within the group. This enables horizontal scaling of message consumption.

```
Topic "click-events" (24 partitions)
  ├── Partition 0  ← assigned to Consumer A
  ├── Partition 1  ← assigned to Consumer A
  ├── Partition 2  ← assigned to Consumer B
  ├── ...
  └── Partition 23 ← assigned to Consumer L

Consumer Group "click-analytics-group" (12 consumers)
  ├── Consumer A ── partitions [0, 1]
  ├── Consumer B ── partitions [2, 3]
  ├── ...
  └── Consumer L ── partitions [22, 23]
```

### How Rebalancing Works

Rebalancing is the mechanism by which Kafka redistributes partitions among consumers when a consumer joins or leaves the group. There are two protocols:

**Eager rebalance (default, `org.apache.kafka.common.rebalancer.EagerRebalance`)**: All consumers in the group stop consuming, revoke all partitions, then the group coordinator reassigns partitions. During this period, **no consumer is processing any messages** — a classic stop-the-world event.

```
1. Consumer C crashes / times out
2. Group coordinator detects failure
3. Coordinator sends "revoke partitions" to ALL consumers
4. All consumers stop processing → STOP THE WORLD
5. Consumers rejoin with a new generation ID
6. Coordinator reassigns partitions
7. Consumers resume processing
```

**Cooperative rebalance (`org.apache.kafka.common.rebalancer.CooperativeRebalance`)**: Consumers only revoke a subset of partitions at a time, and processing continues on unaffected partitions.

### Key Configuration Parameters

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `session.timeout.ms` | 45000 (45s) | Maximum time between heartbeats before the coordinator marks a consumer as dead |
| `heartbeat.interval.ms` | 3000 (3s) | How often the consumer sends heartbeats |
| `max.poll.interval.ms` | 300000 (5min) | Maximum time between two consecutive polls before the coordinator marks the consumer as failed |
| `max.poll.records` | 500 | Maximum number of records returned per poll |
| `rebalance.timeout.ms` | 600000 | Maximum time for consumers to rejoin after a rebalance is triggered |

The relationship between these timeouts forms a critical chain:

```
session.timeout.ms → heartbeat failure detection (fast)
max.poll.interval.ms → processing timeout detection (slow)

Consumer processing ──> poll() interval
     │
     ├── If poll() > max.poll.interval.ms ──> Coordinator removes consumer ──> REBALANCE
     │
     └── If heartbeat > session.timeout.ms ──> Coordinator marks consumer dead ──> REBALANCE
```

## Investigation

### Step 1: Check Consumer Lag

The first sign of trouble was the lag chart spiking. We confirmed it via CLI.

```bash
# Check consumer group lag
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

Partitions 2 and 3 showed **63K and 57K lag** respectively, while partitions 0 and 1 had only 500. The imbalance alone was suspicious — some consumers were clearly unable to keep up.

Run the same command a minute later:

```bash
sleep 60; kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
  --describe --group click-analytics-group | grep -E "LAG|PARTITION" | head -5
```

```
click-analytics-group  click-events    2  13890000  14530000  64000  consumer-2  10.0.0.2
click-analytics-group  click-events    3  13950000  14531000  58100  consumer-2  10.0.0.2
```

Lag was **growing** — not shrinking. This eliminated the possibility of a simple traffic burst that would self-recover.

### Step 2: Inspect Member Changes

Rebalances manifest as consumers joining and leaving the group. Let's check.

```bash
# Describe consumer group with member details
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

Looks healthy now, but the consumer IDs may have changed from the previous check. Run it repeatedly:

```bash
# Run 5 times with 10-second intervals to detect membership changes
for i in $(seq 1 5); do
  echo "=== Check $i ==="
  kafka-consumer-groups --bootstrap-server kafka-cluster:9092 \
    --describe --group click-analytics-group --members --verbose \
    | grep -E "^click-analytics-group"
  sleep 10
done
```

```
=== Check 1 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-ghi-jkl      10.0.0.2/1024 consumer-2  2
...
=== Check 2 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-ghi-jkl      10.0.0.2/1024 consumer-2  2
...
=== Check 3 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-xyz-rst      10.0.0.2/1024 consumer-2  2   ← CONSUMER ID CHANGED
...
=== Check 4 ===
click-analytics-group  consumer-1-abc-def      10.0.0.1/1024 consumer-1  2
click-analytics-group  consumer-2-stu-vwx      10.0.0.2/1024 consumer-2  2   ← CHANGED AGAIN
...
```

The consumer-2 member ID changed three times in 40 seconds. This confirmed **frequent rebalances** — consumer-2 kept leaving and rejoining the group, getting a new member ID each time.

### Step 3: Check Consumer Logs for Rebalance Events

Next, we examined the consumer application logs. The rebalance events are logged at `INFO` level.

```bash
# Search consumer logs for rebalance events
kubectl logs -n analytics --selector=app=click-consumer --tail=1000 | grep -i "rebalance\|revoke\|assign"
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

The log repeats every 30-60 seconds for consumer-2. Key observations:
- Heartbeat timeout causes membership revocation
- Consumer rejoins and gets reassigned
- Cycle repeats

For the other consumers (not the one being kicked out), we saw:

```bash
kubectl logs -n analytics click-consumer-0 --tail=500 | grep -i "rebalance\|revoke"
```

```
2026-06-05 14:02:15 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-1, groupId=click-analytics-group] 
  Revoke previously assigned partitions set(..., ..., ...)
2026-06-05 14:02:18 INFO  o.a.k.c.c.i.ConsumerCoordinator: [Consumer clientId=consumer-1, groupId=click-analytics-group] 
  Assigned to partition(s): ...
```

Every consumer was being forced to revoke partitions and rejoin whenever consumer-2 dropped out. That's the **stop-the-world** behavior of eager rebalance.

### Step 4: JMX Metrics — Rebalance Rate

We checked JMX metrics exposed by the Kafka consumers for the definitive rebalance count.

```bash
# Query JMX for rebalance metrics via jmxterm or prometheus
# If using Prometheus JMX exporter:
curl -s http://localhost:8080/metrics | grep -i rebalance
```

```
# HELP kafka_consumer_coordinator_rebalance_rate_per_hour The number of rebalance operations per hour
# TYPE kafka_consumer_coordinator_rebalance_rate_per_hour gauge
kafka_consumer_coordinator_rebalance_rate_per_hour{client_id="consumer-1",} 34.0
kafka_consumer_coordinator_rebalance_rate_per_hour{client_id="consumer-2",} 34.0
...

# HELP kafka_consumer_coordinator_rebalance_total The total number of rebalance operations
# TYPE kafka_consumer_coordinator_rebalance_total counter
kafka_consumer_coordinator_rebalance_total{client_id="consumer-1",} 127.0
kafka_consumer_coordinator_rebalance_total{client_id="consumer-2",} 127.0
```

**34 rebalances per hour** — that's about one rebalance every 2 minutes. Normal is 0.

### Step 5: Analyze Consumer Configuration

Now we knew the problem was rebalances. The next question: *why* was the consumer being kicked out?

We checked the consumer configuration:

```bash
# Check consumer config via application config file or API
# In a Java application, the config is typically in application.yml or passed as properties
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
        session.timeout.ms: 10000        # 10 seconds ← SUSPICIOUSLY LOW
        heartbeat.interval.ms: 3000      # 3 seconds
        max.poll.interval.ms: 300000     # 5 minutes
        max.poll.records: 500            # 500 records per poll ← TOO HIGH
        max.partition.fetch.bytes: 1048576 # 1 MB
```

Three red flags immediately:

| Config | Value | Kafka Recommended | Problem |
|--------|-------|-------------------|---------|
| `session.timeout.ms` | 10,000 (10s) | 45,000 (45s) | Way too low — JVM GC pauses can easily exceed 10s |
| `max.poll.interval.ms` | 300,000 (5min) | N/A (depends on workload) | Batch processing of 500 Avro records with enrichment took 6-8 min |
| `max.poll.records` | 500 | Depends on processing time | Too many records per poll given the enrichment logic |

## Root Cause

The investigation revealed a cascading failure chain that turned a simple traffic spike into a full-blown outage. Here are the root causes in order:

### 1. `session.timeout.ms=10000` — False Positive Heartbeat Failures

The `session.timeout.ms` was set to 10 seconds (the default in older Spring Kafka versions), while the Kafka 3.4 recommended minimum is 45 seconds.

The consumer application was running on Kubernetes with 4 vCPUs and 8 GB RAM. During peak traffic, the JVM experienced **Young GC pauses of 2-5 seconds**, and occasionally **Full GC pauses of 8-12 seconds** when processing the large Avro records.

A 10-second timeout means that a single Full GC pause of 11 seconds would cause the consumer to miss the heartbeat window. The group coordinator would mark the consumer as dead, triggering a rebalance.

```
Normal:      heartbeat (3s) → heartbeat (3s) → heartbeat (3s) → OK
During GC:   heartbeat (3s) → [GC pause 11s] → ❌ session.timeout.ms=10s exceeded
                                     ↓
             Coordinator marks consumer as DEAD
                                     ↓
                                 REBALANCE
```

### 2. `max.poll.interval.ms=300000` — Batch Processing Exceeds Poll Interval

Even when heartbeats succeeded, the `max.poll.interval.ms` was set to 5 minutes (300,000 ms). With `max.poll.records=500`, each poll returned up to 500 records. The enrichment logic for each record:

1. Deserialize Avro (5-10 ms per record)
2. Enrich with user profile from Redis (20-50 ms per record)
3. Enrich with geo-IP data (10-30 ms per record)
4. Write to down-stream topic (5 ms per record)

Total processing time per record: **40-95 ms**. For 500 records: **20-47 seconds** under normal load.

But under **2x traffic surge**, Redis latency increased (cache miss ratio went up), and individual record processing time ballooned to **200-500 ms**. For 500 records: **100-250 seconds**.

250 seconds is 4.2 minutes — under the 5-minute poll interval. But occasional stragglers (Redis connection retries, network hiccups) pushed individual batch processing to **6-8 minutes**, exceeding the 300,000 ms limit.

When `max.poll.interval.ms` is exceeded, Kafka assumes the consumer is stuck and marks it as failed — triggering yet another rebalance.

```
Normal batch (500 records): 20-47 seconds → within 5 min limit ✓
Peak batch (500 records):   100-250 seconds → barely within limit
Peak batch + hiccup:        360-480 seconds → EXCEEDS 300s limit → REBALANCE
```

### 3. `max.poll.records=500` — Compounding Factor

500 records per poll was appropriate for fast processing, but with the enrichment logic latency variance, it created a high-risk situation. Reducing this would have:
- Reduced per-batch processing time
- Reduced the chance of exceeding `max.poll.interval.ms`
- Reduced GC pressure (fewer records in memory per batch)

### 4. Eager Rebalance Protocol — Stop-the-World Amplification

With the default eager rebalance protocol, every time consumer-2 was kicked out (due to session timeout or poll interval exceeded), **all 12 consumers** stopped processing, revoked their partitions, and rejoined. This created a cascade:

1. Consumer-2 times out → rebalance triggered
2. All 12 consumers stop processing → lag starts growing on ALL partitions
3. After reassignment, consumers resume, but with higher lag
4. Consumer-2 processes backlog → takes longer → exceeds `max.poll.interval.ms` → rebalance again
5. Repeat — the rebalance frequency accelerates because each rebalance increases the processing backlog, which increases processing time, which increases the chance of timeout

```
Initial traffic spike → Consumer-2 slows down
         ↓
    session.timeout.ms exceeded (GC pause)
         ↓
    Eager rebalance (all consumers stop)
         ↓
    Lag grows across all partitions
         ↓
    Consumers struggle to catch up
         ↓
    max.poll.interval.ms exceeded (processing backlog)
         ↓
    Another rebalance → more lag → death spiral
```

This is the **rebalance death spiral** — rebalances beget more rebalances.

## Resolution

### Emergency Fix (Stop the Bleeding)

The immediate priority was to stabilize the consumer group. We adjusted three configuration parameters on the running consumers.

```bash
# Emergency configuration update via environment variables
kubectl set env deployment/click-consumer \
  SPRING_KAFKA_CONSUMER_PROPERTIES_SESSION_TIMEOUT_MS=45000 \
  SPRING_KAFKA_CONSUMER_PROPERTIES_MAX_POLL_INTERVAL_MS=600000 \
  SPRING_KAFKA_CONSUMER_PROPERTIES_MAX_POLL_RECORDS=200
```

```bash
# Rolling restart to pick up changes
kubectl rollout restart deployment/click-consumer
```

Verify the rollout:

```bash
kubectl rollout status deployment/click-consumer
```

```
deployment "click-consumer" successfully rolled out
```

**Config comparison (before vs after)**:

| Parameter | Before | After | Rationale |
|-----------|--------|-------|-----------|
| `session.timeout.ms` | 10,000 (10s) | **45,000 (45s)** | Accommodate GC pauses up to 30s |
| `heartbeat.interval.ms` | 3,000 (3s) | **3,000 (3s)** | No change — keep fast heartbeat detection |
| `max.poll.interval.ms` | 300,000 (5min) | **600,000 (10min)** | Give consumers time to process large batches |
| `max.poll.records` | 500 | **200** | Reduce per-batch processing time and memory pressure |

### Verify Recovery

```bash
# Check lag after stabilization
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

Lag stabilized and started draining. Rebalance rate dropped to 0.

```bash
curl -s http://localhost:8080/metrics | grep "rebalance_rate_per_hour"
```

```
kafka_consumer_coordinator_rebalance_rate_per_hour{client_id="consumer-1",} 0.0
```

### Long-Term Fix: Switch to Cooperative Rebalance

The eager rebalance protocol was amplifying the problem. We switched to cooperative rebalancing (incremental), which only revokes partitions from affected consumers.

```yaml
# application.yml
spring:
  kafka:
    consumer:
      properties:
        # ... other configs
        group.protocol: consumer       # Modern group protocol (Kafka 3.4+)
        partition.assignment.strategy:
          - org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

Or equivalently:

```yaml
spring:
  kafka:
    consumer:
      properties:
        rebalance.protocol: cooperative
```

**Why cooperative rebalance helps**:

| Aspect | Eager (before) | Cooperative (after) |
|--------|----------------|---------------------|
| Behavior | All consumers stop, revoke all, reassign | Only affected partitions revoked |
| Impact on healthy consumers | Stop processing entirely | Continue processing unaffected partitions |
| Time to stabilize | Long (all must rejoin) | Short (minimal disruption) |
| Amplifies cascading failures? | Yes (every rebalance affects all) | No (isolated to affected consumer) |

### Add Monitoring

We added the following alerts to catch this before it becomes an incident again:

| Alert | Threshold | Action |
|-------|-----------|--------|
| Rebalance rate | > 5/hr in any consumer group | Page on-call |
| Consumer lag | > 10,000 for any partition | Page on-call |
| `max.poll.interval.ms` exceeded | Any occurrence in logs | Warning |
| Session timeout | Any occurrence in logs | Warning |
| GC pause time | > 5s (p99) | Investigate |

### Configuration Checklist

Before deploying a Kafka consumer to production, verify these settings:

- [ ] `session.timeout.ms` >= 45000 (or 3x expected max GC pause)
- [ ] `max.poll.interval.ms` >= 2x expected max batch processing time
- [ ] `max.poll.records` = realistic batch size given processing logic
- [ ] Heartbeat interval (`heartbeat.interval.ms`) <= 1/3 of `session.timeout.ms`
- [ ] Prefer cooperative rebalance protocol
- [ ] Monitor rebalance rate and consumer lag
- [ ] Set up JVM GC logging to correlate GC pauses with rebalance events

## Lessons Learned

1. **Default configurations are not safe defaults.** Spring Kafka's `session.timeout.ms=10000` might be fine for dev, but in production with GC pauses, it is a ticking time bomb. Always tune consumer timeouts for your workload's worst-case processing time, not the average.

2. **Rebalances beget more rebalances.** The stop-the-world nature of eager rebalance means that one slow consumer can drag down the entire group. The processing backlog created during the rebalance window makes it more likely that consumers will time out after reassignment, creating a death spiral.

3. **`max.poll.records` is the lever that controls everything.** Instead of trying to tune timeouts around large batches, reduce batch size first. 200 records per poll is often a better starting point than 500. It reduces per-batch latency variance, GC pressure, and the risk of exceeding `max.poll.interval.ms`.

4. **GC pauses and Kafka timeouts are a dangerous combo.** If your consumer does any non-trivial processing (deserialization, enrichment, external calls), monitor GC pause times and ensure `session.timeout.ms` can accommodate the worst case. A 10-second timeout with a JVM that has 8-second Full GC pauses is a disaster waiting to happen.

5. **Cooperative rebalancing should be the default in 2026.** Kafka 3.4+ has had cooperative rebalancing for years. There is almost no reason to use eager rebalance for modern workloads. The incremental nature isolates failure to the affected consumer rather than poisoning the entire group.

6. **Always monitor rebalance rate as a first-class signal.** If your Kafka dashboard doesn't show "rebalances per hour" prominently, add it. A non-zero rebalance rate is almost always a sign of trouble. Zero rebalances is the only acceptable steady state.

## Summary

### Attack Chain

```
Misconfigured Consumer
  │
  ├── session.timeout.ms = 10s (too low)
  ├── max.poll.interval.ms = 5min (too low for peak load)
  └── max.poll.records = 500 (too high)
            │
            ▼
    Traffic Spike (2x normal)
            │
            ▼
    JVM GC pause exceeds 10s
            │
            ▼
    session.timeout.ms exceeded → Consumer marked dead
            │
            ▼
    EAGER REBALANCE (all 12 consumers stop)
            │
            ▼
    Processing lag grows on ALL partitions
            │
            ▼
    Consumers process backlog → exceed max.poll.interval.ms
            │
            ▼
    Another rebalance → LAG GROWS FURTHER
            │
            ▼
    Death Spiral: rebalance → lag → timeout → rebalance
            │
            ▼
    Pipeline latency: 2s → 15min
    Consumer lag: 100 → 500,000+
```

### Recovery Flow

```
Emergency config change
  │
  ├── session.timeout.ms: 10s → 45s
  ├── max.poll.interval.ms: 5min → 10min
  └── max.poll.records: 500 → 200
            │
            ▼
    Rolling restart of consumers
            │
            ▼
    Rebalance rate drops to 0
            │
            ▼
    Lag drains: 500,000 → 500
            │
            ▼
    Processing latency: 15min → 2s
            │
            ▼
    Long-term: cooperative rebalance + monitoring
```

**Bottom line**: Three wrong config values and one traffic spike created a self-sustaining rebalance death spiral that took down a real-time pipeline. The fix was straightforward once the chain was understood — tune timeouts to accommodate real-world processing variance, reduce batch size, and switch to cooperative rebalance to contain failures.
