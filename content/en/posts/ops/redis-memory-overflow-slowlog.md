---
title: "Redis Memory Overflow — How an Unbounded Cache Took Down Our Session Store"
date: 2026-06-08
weight: 100400
slug: "redis-memory-overflow-slowlog"
tags: ["redis", "database", "troubleshooting", "memory", "performance"]
categories: ["Troubleshooting"]
description: "A Redis memory overflow incident — how an application bug caused unbounded key growth, triggering the maxmemory-policy eviction loop, slowlog alerts, and eventually a complete session store failure"
keywords: "redis memory overflow, redis maxmemory, redis eviction policy, redis slowlog, redis oom, redis keyspace analysis"
draft: false
featured: true
cover:
  image: ""
  caption: "Redis Memory Overflow — Troubleshooting"
---

## Common Search Queries

- Redis OOM command not allowed when used memory > maxmemory
- Redis evicted_keys suddenly increasing
- Redis slowlog shows many commands
- Redis used_memory near maxmemory
- redis-cli --bigkeys scan analysis
- Redis session store eviction loop
- Redis maxmemory-policy troubleshooting
- Redis MEMORY DOCTOR report interpretation
- Redis keyspace analysis and cleanup
- Redis CPU 100% slowlog investigation

---

## The Incident

### Environment

| Component | Detail |
|-----------|--------|
| **Redis Version** | 6.2.6 (Standalone, no replication) |
| **Maxmemory** | 8 GB |
| **Eviction Policy** | `allkeys-lru` |
| **Workload** | Session store for Node.js web application |
| **Traffic** | ~20,000 req/s peak |
| **OS** | Ubuntu 20.04, 16 vCPU, 32 GB RAM |
| **Deployment** | Self-managed on EC2 (c5.4xlarge) |

### Timeline

**14:30 — Code Deployment**

A routine backend deployment included a change to the session middleware. A well-intentioned refactor introduced a `sessionId` generator that appended a random UUID on every request. Instead of reusing an existing session ID for the user's session, the application created a brand new key — `session:a1b2c3d4-...` — for virtually every incoming HTTP request.

The code looked something like this (simplified):

```javascript
// Buggy: generates a new session key per request
app.use(session({
  genid: () => uuid.v4(),    // <-- new UUID per request!
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret'
}));
```

**15:10 — First Alert**

PagerDuty fired. Application latency P99 jumped from 50 ms to 2,800 ms. Redis command latency spiked from ~2 ms to 600+ ms.

**15:15 — User Impact**

Users began reporting "Session expired" errors and unexpected logouts. The support ticket queue exploded.

**15:20 — Investigation Begins**

The SRE team connected to the Redis instance and found:

```
OOM command not allowed when used memory > maxmemory
```

Redis had exhausted its 8 GB `maxmemory` and was refusing writes.

---

## Background

### Redis Memory Model

Redis stores all data in-memory. When configured with `maxmemory`, Redis tracks total memory usage and enforces an eviction policy once the limit is reached. Key memory regions include:

- **used_memory**: total bytes allocated by Redis (including overhead)
- **used_memory_rss**: bytes seen by the OS (resident set size)
- **used_memory_peak**: historical peak usage
- **used_memory_lua**: Lua scripting memory
- **used_memory_overhead**: memory used for key-value metadata (not data itself)
- **used_memory_dataset**: actual data memory
- **used_memory_dataset_perc**: percentage of dataset memory relative to net usage
- **maxmemory**: configured hard limit

### Eviction Policies

Redis provides multiple `maxmemory-policy` options:

| Policy | Behavior |
|--------|----------|
| `noeviction` | Return errors on writes when memory limit is reached |
| `allkeys-lru` | Evict least recently used keys from all keys |
| `allkeys-lfu` | Evict least frequently used keys from all keys |
| `volatile-lru` | Evict LRU keys among keys with TTL set |
| `volatile-lfu` | Evict LFU keys among keys with TTL set |
| `allkeys-random` | Evict random keys |
| `volatile-random` | Evict random keys with TTL set |
| `volatile-ttl` | Evict keys with shortest TTL |

The default `noeviction` is safe but may cause write failures. `allkeys-lru`, while common for caching use cases, can be dangerous for session stores — it doesn't distinguish between hot (actively used) sessions and garbage keys.

### Redis Slowlog

`SLOWLOG` records commands whose execution time exceeds a configured threshold (`slowlog-log-slower-than`, default 10,000 microseconds = 10 ms). The log is stored in-memory (configurable with `slowlog-max-len`, default 128 entries).

```
SLOWLOG GET 20
```

Returns entries with:

- **id**: unique entry ID
- **timestamp**: Unix timestamp
- **microseconds**: execution duration
- **command**: the command and its arguments
- **client IP:port**: originating client
- **client name**: optional client name

---

## Investigation

### Step 1: Check Redis Memory Usage

First, we logged into the Redis instance and ran `INFO memory`:

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

**Finding**: `used_memory` was equal to `maxmemory` — the instance was completely full. The fragmentation ratio of 1.00 indicated no significant memory fragmentation (at this point, nearly the entire allocation was actively used).

### Step 2: Check Eviction Stats

```
$ redis-cli INFO stats | grep evicted

evicted_keys:2147593
evicted_keys_per_second:298.33
```

**Finding**: Over 2.1 million keys had been evicted. At peak, Redis was evicting ~300 keys per second. This explained the high CPU — each eviction involves LRU sampling, key removal, and freeing memory, which is CPU-intensive, especially under sustained write pressure.

### Step 3: Check Slowlog

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

**Findings**:
- `SETEX` commands for `session:anon:*` keys took 342 ms (far above the default 10 ms threshold)
- `GET` commands for legitimate session keys took 289 ms — normally these take <1 ms
- The slowlog was dominated by `SETEX` for anonymous session keys and `GET` for authenticated session keys

The `SETEX` slowness was expected under memory pressure (eviction + allocation churn), but the `GET` slowness was alarming — it suggested the entire Redis was performing poorly, not just write-heavy operations.

### Step 4: Analyze Keyspace

```
$ redis-cli INFO keyspace

# Keyspace
db0:keys=2187642,expires=128,avg_ttl=0
```

**Finding**: 2.18 million keys in database 0, but only 128 keys had TTLs set. The `avg_ttl` of 0 confirmed that the overwhelming majority of keys were **permanent** — they would never expire on their own.

### Step 5: Find Key Patterns

```
$ redis-cli SCAN 0 MATCH session:* COUNT 10000

1) "235672"
2) 1) "session:anon:a1b2c3d4-e5f6-7890-abcd-ef1234567890"
   2) "session:anon:b2c3d4e5-f6a7-8901-bcde-f12345678901"
   3) "session:auth:user-54321"
   ...
```

Even with `COUNT 10000`, the `SCAN` took multiple seconds to complete. The key distribution was clear:

- **`session:anon:*`**: ~2.1 million keys — these were the buggy keys created per-request
- **`session:auth:*`**: ~80,000 keys — legitimate authenticated session keys

The ratio was ~26:1 garbage-to-legitimate keys.

### Step 6: MEMORY STATS and MEMORY DOCTOR

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

`MEMORY DOCTOR` provided a concise, human-readable diagnosis confirming our suspicions.

### Step 7: Check Client Connections

```
$ redis-cli CLIENT LIST

id=43921 addr=192.168.1.50:43802 fd=31 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=2056 events=r cmd=SETEX user=default
id=43922 addr=192.168.1.51:43901 fd=32 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=2056 events=r cmd=GET user=default
id=43923 addr=192.168.1.52:44012 fd=33 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=0 argv-mem=0 obl=0 oll=0 omem=0 tot-mem=2056 events=r cmd=SETEX user=default
...
```

**Finding**: Dozens of connections with `cmd=SETEX` and `cmd=GET` piling up. Some connections showed `omem` (output memory) values that were growing — Redis was struggling to keep up with the write load, causing output buffers to accumulate.

### Step 8: redis-cli --bigkeys

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

**Finding**: Nearly all keys were strings (session data). The total keyspace was 2.18M keys using ~4.9 GB of data. The average key size was 2.26 KB. This confirmed that the problem was not large individual keys, but an **enormous number of keys** — a classic "small-key flood" problem.

---

## Root Cause

The root cause chain was:

```
Code Bug → Per-request session keys → 2.1M garbage keys in 2 hours
→ Redis maxmemory exhausted → allkeys-lru eviction loop starts
→ Legitimate sessions evicted → Users redirected to login
→ More login requests → More garbage session keys → More evictions
→ CPU saturation (100%) → All Redis commands slow (including GET)
→ Complete session store degradation
```

### The Bug

The application code used `uuid.v4()` as the session ID generator **without checking whether a session already existed for that client**. Every HTTP request, whether from a new user or an existing user, generated a new session key in Redis:

```javascript
// Problematic pattern
app.use(session({
  genid: () => uuid.v4(), // New key on every request!
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret',
  resave: false,
  saveUninitialized: false  // This was supposed to help but didn't
}));
```

The `resave: false` and `saveUninitialized: false` settings should have prevented saving sessions that weren't modified. However, a middleware ordering issue in the deployment caused the session middleware to run before authentication, creating anonymous sessions (`session:anon:*`) for all unauthenticated requests — including API health checks, favicon requests, and bot traffic.

### Why the Eviction Policy Was Wrong

`allkeys-lru` evicts the **least recently used** keys across the entire keyspace. Under normal cache workloads, this makes sense. For a session store:

1. Garbage keys (`session:anon:*`) were **never accessed again** after creation — making them prime LRU eviction candidates
2. But Redis evicts in batches, and the eviction algorithm uses sampling, not a global LRU
3. When the garbage keys outnumbered legitimate keys 26:1, even with random sampling, Redis would often sample only garbage keys — and keep legitimate sessions
4. However, the sustained write rate (thousands of new keys per second) meant Redis was **constantly evicting**, consuming significant CPU just on eviction decisions

The real problem: the `allkeys-lru` policy combined with the write storm caused **eviction-loop-induced CPU saturation**. Redis became so busy evicting that legitimate commands queued behind eviction work, causing the multi-second latencies.

### Missing Protections

| Protection | Status | Impact |
|------------|--------|--------|
| Key TTL | Missing | Garbage keys never auto-expired |
| Maxmemory alerting | Missing | Team didn't know memory was filling up |
| Keyspace monitoring | Missing | Key count growth went unnoticed |
| Slowlog alerting | Misconfigured | Slowlog was enabled but no alerts fired |
| Session key lifecycle review | Missing | Code review missed the bug |
| Connection pooling limits | Inadequate | App servers opened too many Redis connections |

---

## Resolution

### Emergency: Stop the Bleeding

**Step 1: Batch delete garbage keys**

```bash
# Delete anonymous session keys in batches using SCAN + pipelining
redis-cli -n 0 --scan --pattern "session:anon:*" | head -500000 | \
  xargs -L 1000 redis-cli DEL

# Repeat until the garbage keys are gone
redis-cli -n 0 --scan --pattern "session:anon:*" | wc -l  # Check remaining
```

We used `head -500000` to avoid overwhelming the Redis connection. Each batch of 1,000 keys (using `xargs -L 1000`) took ~3 seconds initially, improving as memory pressure decreased.

**Step 2: Count remaining garbage keys**

```bash
# Count all session keys
redis-cli KEYS "session:*" | wc -l

# Count specific patterns
redis-cli KEYS "session:anon:*" | wc -l
redis-cli KEYS "session:auth:*" | wc -l
```

> **Warning**: Do NOT use `KEYS` in production on a large keyspace — it blocks Redis for the duration. We used it here only under emergency conditions after verifying the instance was already saturated. Prefer `SCAN` for normal operations.

**Step 3: Change eviction policy temporarily**

```bash
# Stop eviction loop
redis-cli CONFIG SET maxmemory-policy noeviction
```

This prevented new writes from triggering evictions. With `noeviction`, writes that would exceed `maxmemory` are rejected with the OOM error — which is less harmful than an infinite eviction loop. Application errors are preferable to total session store breakdown.

**Step 4: Restart affected app servers**

After deleting garbage keys, we restarted the application servers to clear their local session caches and force fresh connections to Redis.

### Short-term: Set TTL on Session Keys

```bash
# Set TTL on all authenticated session keys (24 hours)
redis-cli -n 0 --scan --pattern "session:auth:*" | \
  xargs -L 1000 redis-cli EXPIRE 86400
```

This ensured that even legitimate sessions would eventually expire if users stopped using them, preventing future accumulation.

```bash
# Verify TTLs are set
redis-cli INFO keyspace
# Expected: db0:keys=82345,expires=82345,avg_ttl=...
```

### Long-term: Application Fix

The application code was fixed:

```javascript
// Fixed: uses a persistent session store with stable session IDs
app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: 'my-secret',
  name: 'sid',             // Custom cookie name
  cookie: { secure: true, httpOnly: true, maxAge: 86400000 },
  resave: false,
  saveUninitialized: false // Don't save empty sessions
}));

// Session ID is auto-generated by express-session once per session,
// NOT per request. The genid option was removed entirely.
```

### Monitoring Additions

**Memory alerting (Prometheus + Alertmanager):**

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
          summary: "Redis memory usage > 80%"
          
      - alert: RedisMemoryCritical
        expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis memory usage > 95% — eviction imminent"
          
      - alert: RedisEvictionsHigh
        expr: rate(redis_evicted_keys_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Redis evicting > 10 keys/sec"
```

**Slowlog configuration:**

```bash
# Configure slowlog
redis-cli CONFIG SET slowlog-log-slower-than 10000  # 10 ms threshold
redis-cli CONFIG SET slowlog-max-len 1024           # Keep more entries
redis-cli SLOWLOG RESET                              # Clear old entries
```

**Keyspace monitoring:**

```bash
# Simple key count monitoring script
#!/bin/bash
COUNT=$(redis-cli DBSIZE)
THRESHOLD=500000
if [ "$COUNT" -gt "$THRESHOLD" ]; then
  echo "ALERT: Redis keyspace has $COUNT keys (threshold: $THRESHOLD)"
  redis-cli INFO keyspace | grep -E "^db0"
fi
```

### Tuning maxmemory-samples

The `maxmemory-samples` setting controls the LRU sample size (default 5). Larger values approximate true LRU more closely but cost more CPU per eviction:

```bash
# Default (balanced between accuracy and CPU)
redis-cli CONFIG SET maxmemory-samples 5

# For eviction-sensitive workloads
redis-cli CONFIG SET maxmemory-samples 10
```

In our case, tuning this wouldn't have prevented the incident, but it's worth noting for future optimization.

### Consider Redis Cluster

For long-term scalability, Redis Cluster provides:

- **Automatic sharding** across multiple nodes (16,384 hash slots)
- **Distributed memory** — no single-node maxmemory bottleneck
- **High availability** with replica nodes and failover
- **Linear scalability** — add nodes to increase capacity

Migration to Redis Cluster requires application changes (cluster-aware client), but for a session store handling 20K RPM, it's a worthwhile investment.

---

## Lessons Learned

### What Went Well

- **Monitoring detected the anomaly** within minutes of the deploy
- **Slowlog provided immediate visibility** into which commands were affected
- **MEMORY DOCTOR** gave a concise, actionable diagnosis
- **Existing runbook patterns** for key cleanup were reusable
- **Team communicated early** to stakeholders about the incident

### What Went Wrong

1. **No maxmemory alerting**: The biggest gap. Memory filled from 40% to 100% in under 2 hours with no notification.
2. **Missing TTL on session keys**: A fundamental best practice was violated. Every key in Redis should have a TTL unless there is an explicit reason not to.
3. **Inappropriate eviction policy for session store**: `allkeys-lru` is designed for caching, not for session storage where data loss means user disruption.
4. **Code review missed the session key bug**: The `genid: () => uuid.v4()` pattern looked innocent in isolation. A session-flow-focused review would have caught it.
5. **No keyspace growth alarms**: Key count doubling from 80K to 2.1M should have triggered automated response.
6. **Slowlog alerts misconfigured**: Slowlog was enabled but the logs weren't shipped to the monitoring system. They were only useful for manual investigation.

### Key Takeaways

| Area | Takeaway |
|------|----------|
| **TTL Discipline** | Every Redis key must have a TTL. Enforce at the application layer and monitor keys without TTLs. |
| **Eviction Policy** | Session stores should use `volatile-ttl` or `volatile-lru`, not `allkeys-*`, to protect data. |
| **Alerting** | Alert on `used_memory / maxmemory > 0.8` and `evicted_keys > 0`. |
| **Capacity Planning** | Monitor key count growth trends, not just memory usage. |
| **Code Review** | Session lifecycle logic deserves dedicated review focus. |
| **Graceful Degradation** | `noeviction` with application-level OOM handling is better than eviction loop chaos. |

---

## Summary

### Incident Timeline

```
14:30  Code deployed — per-request session keys begin flooding Redis
14:45  Redis memory climbs past 6 GB (75%)
15:00  Memory hits 7.5 GB (94%), eviction begins
15:05  Eviction loop saturates CPU, slowlog fills
15:10  PagerDuty alerts — latency spike detected
15:15  Users report session errors, logouts
15:20  SRE investigation begins — OOM errors found
15:25  Emergency key deletion starts
15:40  Garbage keys removed, eviction stopped
15:45  noeviction policy applied, app servers restarted
16:00  Service restored to normal
16:30  TTL applied to remaining session keys
17:00  Monitoring and alerting improvements deployed
```

### Flow Diagram

```
                    ┌──────────────────────┐
                    │  Code Bug: per-req   │
                    │  session key creation│
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ 2.1M garbage keys    │
                    │ in 2 hours           │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Redis maxmemory      │
                    │ = 8 GB (100%)        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ allkeys-lru eviction  │
                    │ loop begins           │
                    └──────────┬───────────┘
                               │
                    ┌──────────┴──────────┐
                    ▼                     ▼
            ┌──────────────┐   ┌──────────────────┐
            │ Legitimate   │   │ CPU 100% — all   │
            │ sessions     │   │ commands slow     │
            │ evicted      │   │ (incl. GET)       │
            └──────┬───────┘   └────────┬─────────┘
                   │                    │
                   ▼                    ▼
            ┌──────────────┐   ┌──────────────────┐
            │ Users        │   │ Slowlog fills    │
            │ logged out   │   │ with >300ms cmds │
            └──────┬───────┘   └────────┬─────────┘
                   │                    │
                   └──────┬─────────────┘
                          ▼
               ┌──────────────────────┐
               │ Complete session     │
               │ store degradation    │
               └──────────────────────┘
```

### Quick Reference Commands

```bash
# Check memory
redis-cli INFO memory | grep -E "used_memory_human|maxmemory_human|maxmemory_policy"

# Check evictions
redis-cli INFO stats | grep evicted_keys

# Check slowlog
redis-cli SLOWLOG GET 20

# Analyze keyspace
redis-cli INFO keyspace
redis-cli MEMORY STATS
redis-cli MEMORY DOCTOR

# Find key patterns
redis-cli --scan --pattern "session:*" | head -100

# Check clients
redis-cli CLIENT LIST

# Big keys scan (use with caution)
redis-cli --bigkeys

# Batch delete keys
redis-cli --scan --pattern "prefix:*" | xargs -L 1000 redis-cli DEL

# Set TTL on key pattern
redis-cli --scan --pattern "session:*" | xargs -L 1000 redis-cli EXPIRE 86400

# Change eviction policy (temporary)
redis-cli CONFIG SET maxmemory-policy noeviction

# Configure slowlog
redis-cli CONFIG SET slowlog-log-slower-than 10000
redis-cli CONFIG SET slowlog-max-len 1024
```

---

*Redis 6.2 * 8 GB maxmemory * 20K RPM * 2.1M keys in 2 hours * allkeys-lru eviction loop * Session store failure *

> This article is part of the production troubleshooting series. Have you encountered a Redis eviction loop before? Share your story in the comments below.
