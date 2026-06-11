---
title: "MongoDB Replica Set Election — When the Primary Stepped Down and Nobody Won the Vote"
date: 2026-06-11
weight: 100510
slug: "mongodb-replica-election-failure"
tags: ["mongodb", "database", "troubleshooting", "replication", "election"]
categories: ["Troubleshooting"]
description: "A MongoDB replica set election failure incident — how a networking latency spike triggered a false primary step-down, and the subsequent election failed because of an uneven voting count and a stale secondary, leaving the cluster without a primary for 11 minutes"
keywords: "mongodb replica set election, mongodb no primary, mongodb election timeout, mongodb replication lag, mongodb rs.status"
draft: false
featured: true
cover:
  image: ""
  caption: "MongoDB Replica Set Election — Troubleshooting"
---

# MongoDB Replica Set Election — When the Primary Stepped Down and Nobody Won the Vote

## Common Search Queries

| English | Chinese |
|---------|---------|
| MongoDB replica set election failed | MongoDB 复制集选举失败 |
| MongoDB no primary after election | MongoDB 选举后无主节点 |
| MongoDB electionTimeoutMillis too low | MongoDB electionTimeoutMillis 设置过低 |
| MongoDB arbiter not voting in election | MongoDB 仲裁节点不投票 |
| MongoDB heartbeat failed step down | MongoDB 心跳失败导致主节点降级 |
| MongoDB rs.status() no PRIMARY | MongoDB rs.status() 无 PRIMARY |
| MongoDB secondary stale oplog election | MongoDB 从节点 oplog 落后选举失败 |
| MongoDB force reconfig after election failure | MongoDB 选举失败强制重新配置 |
| MongoDB replication lag causes election failure | MongoDB 复制延迟导致选举失败 |
| MongoDB cross-AZ replica set election tuning | MongoDB 跨可用区复制集选举调优 |

---

## The Incident

### Environment

| Component | Specification |
|-----------|---------------|
| MongoDB Version | 7.0.14 (Community) |
| Replica Set Size | 3 nodes (Primary + Secondary + Secondary) |
| Arbiter | 1 arbiter (votes only, no data) |
| Write Concern | w: "majority" |
| Read Preference | primary (default) |
| Deployment | Single-region, 3 availability zones |
| Network RTT (normal) | ~1-2 ms |
| Storage Engine | WiredTiger |
| Oplog Size | 10 GB (default for 7.0) |

### Timeline

| Time (UTC) | Event |
|------------|-------|
| 14:03:00 | Network maintenance window begins |
| 14:03:12 | Latency spike: RTT between nodes jumps from 2 ms to 800 ms |
| 14:03:15 | Primary detects heartbeat timeout from both secondaries |
| 14:03:16 | Primary steps down, transitions to SECONDARY |
| 14:03:17 | Election triggered — both secondaries stand for election |
| 14:03:19 | Election round 1 fails — neither secondary has caught-up oplog |
| 14:03:30 | Election round 2 fails — same reason, timeout |
| 14:04:00 - 14:13:30 | Continuous election retries — all fail |
| 14:13:45 | Secondary B catches up on oplog, wins election, becomes PRIMARY |
| 14:14:00 | Application reconnects, writes resume |
| 14:14:30 | Original primary rejoins as SECONDARY |

**Downtime: 11 minutes** — all writes to the cluster failed during this period.

### Symptoms

- Application teams reported **HTTP 500 errors** across all write endpoints
- MongoDB driver logs showed repeated `NotWritablePrimary` / `not primary` errors
- `rs.status()` displayed `stateStr: "SECONDARY"` for all three data nodes
- The arbiter showed `stateStr: "ARBITER"` but could not break the tie unilaterally
- `rs.isMaster()` returned `"ismaster": false` for every node
- The MongoDB logs were filled with `Election failed, sleeping` and `heartbeat failed` messages
- Alertmanager fired `MongoDB_NoPrimary` alert (silenced during maintenance window)

---

## Background

### MongoDB Replication Architecture

MongoDB replica sets use an **oplog** (operations log) to replicate data. Every write on the primary is recorded in the primary's oplog. Secondaries copy these operations asynchronously and apply them to their own data files. This is analogous to MySQL's binary log replication but operates at the document level.

Key concepts:

- **Oplog**: A capped collection (`local.oplog.rs`) that stores all write operations. Each secondary maintains its own oplog by replaying operations from the primary (or another secondary).
- **Replication lag**: The time difference between the last operation applied on a secondary and the last operation on the primary. Measured by comparing `optimeDate` values in `rs.status()`.
- **Heartbeat**: Each member sends a heartbeat (ping) to every other member every 2 seconds by default (`heartbeatIntervalMillis: 2000`). Heartbeats carry state information including the member's current term and optime.
- **Term**: A monotonically increasing number representing a leadership period. Each election increments the term.

### The Election Process

When a replica set needs to choose a new primary (either because the current primary stepped down, became unreachable, or a higher-priority member joined), the following happens:

1. **Vote Request**: Any secondary that detects it cannot reach the primary initiates an election by sending a `voteRequest` to all other members.
2. **Vote Response**: Each member votes for the candidate it considers most suitable. Suitability is determined by:
   - **Priority**: Higher priority members are preferred.
   - **Oplog freshness**: A secondary whose oplog is behind the voter's own oplog will not receive a vote.
   - **Term**: Members only vote once per term.
3. **Quorum**: A candidate needs a majority of votes (ceil(N/2)) to become primary. For a 3-node set + arbiter, that is 2 votes out of 3 voting members (arbiter votes but has no data).
4. **Winning**: If no candidate reaches a majority, the election fails and retries after a random delay.

### Election Timing

| Setting | Default | Description |
|---------|---------|-------------|
| `electionTimeoutMillis` | 10000 (10s) | Max time a secondary waits to detect primary failure before starting an election |
| `heartbeatIntervalMillis` | 2000 (2s) | Interval between heartbeat pings |
| `heartbeatTimeoutSecs` | 10 | Seconds before a heartbeat is considered failed |

When `electionTimeoutMillis` expires without hearing from the primary, a secondary assumes the primary is down and calls an election. If this value is set too low, the cluster becomes **jittery** — transient network blips trigger unnecessary elections.

### The Arbiter Role

An arbiter is a lightweight member that **participates in elections but does not hold data**. It exists solely to provide a tie-breaking vote in even-numbered replica sets (e.g., 2 data nodes + 1 arbiter). Key rules:

- An arbiter can vote but **cannot become primary**
- An arbiter does not replicate data and has no oplog
- An arbiter is typically deployed on a separate, low-resource host
- An arbiter's vote counts toward the majority threshold

---

## Investigation

The following steps were taken during the incident to diagnose the cluster state.

### 1. Check Replica Set Status

```javascript
rs.status()
```

The output revealed that all three data nodes were in `SECONDARY` state:

```json
{
  "set": "rs0",
  "members": [
    { "_id": 0, "name": "node-a:27017", "stateStr": "SECONDARY", "uptime": 84321 },
    { "_id": 1, "name": "node-b:27017", "stateStr": "SECONDARY", "uptime": 84321 },
    { "_id": 2, "name": "node-c:27017", "stateStr": "SECONDARY", "uptime": 84321 },
    { "_id": 3, "name": "arbiter:27017", "stateStr": "ARBITER", "uptime": 84321 }
  ],
  "ok": 1
}
```

No PRIMARY existed. The arbiter was online but could not vote in a way that broke the tie because both secondaries had equal priority.

### 2. Check Current Primary

```javascript
rs.isMaster()
```

All nodes returned `"ismaster": false`. This confirmed there was no writable node in the cluster.

### 3. Check Election Count

```javascript
rs.status().electionParticipantMetrics
```

This showed the number of election attempts. In our case, over 30 election rounds had been attempted in 11 minutes.

### 4. Check Replication Lag and Oplog Window

```javascript
rs.printSecondaryReplicationInfo()
```

Output on both secondaries showed significant lag:

```
source: node-b:27017
    syncedTo: '2026-06-11T14:03:15.123Z'
    lag: 45s (estimated)
source: node-c:27017
    syncedTo: '2026-06-11T14:03:20.456Z'
    lag: 40s (estimated)
```

The secondaries were 40-45 seconds behind the former primary at the time of step-down. This lag was caused by a **bulk write operation** (batch import of ~500k documents) that was still being applied when the network spike hit.

Checking the oplog window:

```javascript
rs.printReplicationInfo()
```

```
configured oplog size:   10240 MB
log length start to end: 2026-06-11T13:45:00Z to 2026-06-11T14:14:00Z
oplog first event time:  2026-06-11T12:00:00Z
oplog last event time:   2026-06-11T14:14:00Z
now:                     2026-06-11T14:14:30Z
```

The oplog window was healthy (~2 hours), so the oplog had not been truncated.

### 5. Check Election Logs in mongod Log File

```bash
grep -i "election\|stepdown\|term" /var/log/mongodb/mongod.log | tail -30
```

Key log lines found:

```
2026-06-11T14:03:16.123Z I REPL [replexec-0] Starting an election, term: 7
2026-06-11T14:03:16.456Z I REPL [replexec-0] VoteRequester(term:7) received a yes vote from node-b; response message: { term: 7, voteGranted: true }
2026-06-11T14:03:16.457Z I REPL [replexec-0] VoteRequester(term:7) received a no vote from node-c; reason: candidate's optime is behind mine
2026-06-11T14:03:16.458Z I REPL [replexec-0] Election failed, term: 7, candidate: node-b, reason: insufficient votes (1 of 2 needed)
2026-06-11T14:03:19.001Z I REPL [replexec-0] Starting an election, term: 8
2026-06-11T14:03:19.234Z I REPL [replexec-0] VoteRequester(term:8) received a no vote from node-b; reason: candidate's optime is behind mine
2026-06-11T14:03:19.235Z I REPL [replexec-0] VoteRequester(term:8) received a no vote from node-c; reason: candidate's optime is behind mine
2026-06-11T14:03:19.236Z I REPL [replexec-0] Election failed, term: 8, candidate: node-a, reason: insufficient votes (0 of 2 needed)
```

The logs show a repeating pattern: each secondary refused to vote for the other because each believed its own oplog was ahead. Neither could achieve a majority.

### 6. Check Heartbeat History

```bash
grep -i "heartbeat" /var/log/mongodb/mongod.log | tail -20
```

```
2026-06-11T14:03:12.789Z I REPL [replexec-0] Heartbeat failed after 10 retries to node-b:27017; RS102: heartbeat timeout
2026-06-11T14:03:12.790Z I REPL [replexec-0] Heartbeat failed after 10 retries to node-c:27017; RS102: heartbeat timeout
2026-06-11T14:03:13.001Z I REPL [replexec-0] Heartbeat failed again to node-b:27017; RS102
2026-06-11T14:03:13.002Z I REPL [replexec-0] Heartbeat failed again to node-c:27017; RS102
```

The 800ms RTT latency caused heartbeat responses to arrive well past the 2-second interval window, triggering the timeout.

### 7. Check Voting Configuration

```javascript
rs.conf().members
```

```json
[
  { "_id": 0, "host": "node-a:27017", "priority": 1, "votes": 1 },
  { "_id": 1, "host": "node-b:27017", "priority": 1, "votes": 1 },
  { "_id": 2, "host": "node-c:27017", "priority": 1, "votes": 1 },
  { "_id": 3, "host": "arbiter:27017", "priority": 0, "votes": 1 }
]
```

All three data nodes had equal priority (1). This is the default configuration, but in a 3+1 topology, equal priorities mean no member has an advantage during elections. The arbiter's priority is 0 (it cannot become primary), but its vote counts.

```javascript
rs.conf().settings
```

```json
{
  "electionTimeoutMillis": 2000,
  "heartbeatIntervalMillis": 2000,
  "catchUpTimeoutMillis": 3000
}
```

`electionTimeoutMillis` was set to **2000 ms** (the minimum allowed). This is extremely aggressive. The default is 10000 ms. The team had lowered it to accelerate failover during planned maintenance, but the side effect was that even minor network jitter triggered elections.

### 8. Check Network Latency Between Nodes

```bash
ping -c 10 node-b
```

```
PING node-b (10.0.2.101) 56(84) bytes of data.
64 bytes from 10.0.2.101: icmp_seq=1 ttl=64 time=782 ms
64 bytes from 10.0.2.101: icmp_seq=2 ttl=64 time=803 ms
64 bytes from 10.0.2.101: icmp_seq=3 ttl=64 time=791 ms
...
--- node-b ping statistics ---
10 packets transmitted, 10 received, 0% packet loss, time 10005ms
rtt min/avg/max/mdev = 782/798/812/10.234 ms
```

Round-trip time was ~800 ms — no packet loss but very high latency. This confirmed a network issue (later traced to a misconfigured switch during the maintenance window).

### 9. Check serverStatus Metrics

```javascript
db.adminCommand({serverStatus:1}).replication
```

The `serverStatus` output on each secondary showed:

```json
{
  "replication": {
    "execution": {
      "candidate": false,
      "master": false,
      "primary": false
    },
    "electionMetrics": {
      "numElections": 34,
      "numElectionsAborted": 0,
      "numElectionsFailed": 33,
      "numElectionsWon": 1
    }
  }
}
```

Over 33 failed election attempts and only 1 win (the final one that restored the cluster).

---

## Root Cause

The root cause chain involves three intersecting failures:

### Trigger: Network Latency Spike

During a scheduled network maintenance window, a switch misconfiguration caused the RTT between replica set members to jump from ~2 ms to ~800 ms. This is within the range of a typical cross-continent link, not a local network. The elevated latency caused heartbeat requests to exceed their timeout.

### Failure 1: Primary Step-Down

The primary (node-a) detected heartbeat failures from both secondaries (node-b, node-c). Since the default `heartbeatTimeoutSecs` is 10 seconds and `heartbeatIntervalMillis` is 2000 ms, the primary marked both secondaries as unreachable and stepped down. This was a **legitimate self-preservation** behavior — a primary that cannot communicate with a majority of voting members should step down to prevent split-brain.

### Failure 2: Election Deadlock

Both secondaries had **replication lag of 40-45 seconds** due to an ongoing bulk write operation (batch import of ~500k documents). When the election started, each secondary checked the other's oplog freshness. Node-b's oplog was at timestamp T1; node-c's oplog was at timestamp T2 (slightly behind). Neither could vote for the other because:

- Node-b would not vote for node-c because node-c's oplog was behind node-b's.
- Node-c would not vote for node-b because node-b's optime was in a **different term** from node-c's perspective (due to the step-down and election cycle).

The arbiter voted for whoever asked first, but the candidate still needed **2 votes** (majority of 3 voting members) — and could only get 1 (itself + arbiter, but arbiter's vote makes 1 out of 2 needed).

### Failure 3: Aggressive electionTimeoutMillis

```javascript
settings.electionTimeoutMillis = 2000
```

The team had set this to the minimum value (2000 ms) thinking it would speed up failover. The unintended consequence:

- When an election failed, the next secondary waited only 2 seconds before calling a new election
- The catch-up phase (`catchUpTimeoutMillis` was 3000 ms) was too short for the secondaries to fetch the backlogged oplog entries under 800 ms latency
- Each election incrementing the term made it harder for lagging secondaries to participate

### Why 11 Minutes?

Each election round took roughly 2-3 seconds (election timeout + vote exchange + failure detection). With 33 failed rounds, the cumulative time was:

- ~3 seconds per round x 33 rounds = ~99 seconds of election overhead
- Plus the time spent in catch-up attempts (~50 seconds total)
- Plus the backoff between retries (exponential: 1s, 2s, 4s, 8s...)
- The real bottleneck: the secondaries needed ~600 seconds (10 minutes) to clear the replication backlog under 800 ms latency

The bulk write had produced ~500 MB of oplog entries. At 800 ms RTT, the sustained throughput for oplog sync dropped to roughly 1-2 MB/s, versus the normal ~50 MB/s. It simply took 10 minutes to drain the backlog.

### Summary Diagram

```
  Latency Spike (2ms -> 800ms)
           |
    Heartbeat Timeout
           |
  Primary Steps Down
           |
  Election Triggered
    /            \
   /              \
Node-b has lag   Node-c has lag
   \              /
    \            /
  Neither can get majority vote
           |
   electionTimeoutMillis=2s
           |
  Premature timeouts, retry loop
           |
  11 minutes of no primary
```

---

## Resolution

### Emergency Recovery (During Incident)

When the cluster has no primary, you have two options.

**Option 1: Force reconfig — prioritize a specific secondary**

```javascript
cfg = rs.conf()
cfg.members[1].priority = 10   // node-b gets highest priority
cfg.members[2].priority = 1    // node-c lower priority
cfg.members[3].priority = 0    // arbiter stays at 0
rs.reconfig(cfg, {force: true})
```

The `{force: true}` flag allows reconfig even when a majority of nodes are not healthy. This forces the cluster to converge on node-b as the new primary. Node-b will catch up the other nodes after becoming primary.

**Option 2: Manually step up a secondary**

On the stepped-down primary:

```javascript
rs.stepDown(300)
```

This tells the current node (in SECONDARY state) to not stand for election for 300 seconds, giving another node a chance to win. However, in our case this did not work because the issue was not about the old primary blocking elections — it was that neither secondary could get votes.

**Option 3: Restart mongod on one secondary with a higher priority config**

If reconfig is not possible (e.g., network partition prevents the command from reaching all nodes), restarting the mongod process on a specific secondary with a modified config file that sets `priority: 10` can force it to become primary.

**What Actually Worked**

In this incident, the resolution was **time**. After approximately 10 minutes, node-b finished applying the bulk write operations from its oplog buffer. Once its `optime` caught up, it received a vote from node-c and the arbiter, won the election with 2 votes, and became primary.

The team then applied the configuration fixes (below) to prevent a recurrence.

### Immediate Configuration Fix

```javascript
cfg = rs.conf()
cfg.settings.electionTimeoutMillis = 10000   // changed from 2000 ms to 10000 ms
cfg.members[0].priority = 3                  // node-a: preferred primary
cfg.members[1].priority = 2                  // node-b: first failover target
cfg.members[2].priority = 1                  // node-c: second failover target
cfg.members[3].priority = 0                  // arbiter: stays arbiter
rs.reconfig(cfg)
```

After reconfig:

```javascript
rs.conf().settings.electionTimeoutMillis
// 10000

rs.conf().members.map(m => ({host: m.host, priority: m.priority}))
// [
//   {host: "node-a:27017", priority: 3},
//   {host: "node-b:27017", priority: 2},
//   {host: "node-c:27017", priority: 1},
//   {host: "arbiter:27017", priority: 0}
// ]
```

### Long-Term Remediation

| Category | Action | Details |
|----------|--------|---------|
| Configuration | Increase `electionTimeoutMillis` | Set to 10000 ms (default) for cross-AZ deployments. Even 15000 ms is reasonable for networks with >10 ms RTT. |
| Configuration | Set explicit priorities | Use 5, 3, 1 (or similar) to prevent ties. Never leave all members at priority 1 in an even-numbered voter set. |
| Configuration | Tune `catchUpTimeoutMillis` | Increase to 10000 ms if replication lag is common during bulk operations. |
| Application | Pin write concern | Always use `w: "majority"` for production writes. Avoid `w: 1` (unacknowledged). |
| Application | Bulk write throttling | Rate-limit bulk operations to prevent oplog backlog. Use `maxTimeMS` and batch splitting. |
| Monitoring | Track election metrics | Alert on `mongodb_rs_state != 1` (primary). Monitor `mongodb_rs_member_optime_date` for lag. |
| Monitoring | Secondary lag alert | Alert if any secondary's lag exceeds 30 seconds. This is the canary for election failures. |
| Monitoring | Heartbeat metrics | Graph `mongodb_rs_member_heartbeat_latency` to detect network jitter early. |
| Infrastructure | Chaos engineering | Run scheduled network partition tests using `tc` (traffic control) or a service mesh. |
| Infrastructure | Consider 5-node replica set | A 5-node set (3 data + 2 data, or 3 data + 2 arbiters) provides higher fault tolerance and avoids even-voter tie issues. |
| Driver | Driver writeConcern | Set `writeConcern: "majority"` at the MongoDB driver level, not just the application logic level. |

### Verification After Fix

After applying the fix, verify the cluster is healthy:

```javascript
rs.status().members.forEach(m => {
  print(m.name + ": " + m.stateStr + " | optime: " + m.optimeDate);
})
```

Output should show one PRIMARY, two SECONDARYs, and one ARBITER, all with close optime dates.

```bash
# Simulate a clean election test using failpoint (development only)
mongosh --eval "db.adminCommand({replSetTest: {name: 'rs0', forceElection: true}})"
```

---

## Lessons Learned

### What Went Well

- The arbiter remained accessible throughout the incident, preventing a full cluster outage (the cluster had a quorum, just no candidate could reach a majority).
- Application-level retries and circuit breakers prevented a cascading failure to downstream services.
- The maintenance window was scheduled during low traffic, so the blast radius was limited.
- MongoDB's election retry mechanism eventually worked — it just took much longer than expected.

### What Went Wrong

- **electionTimeoutMillis = 2000 ms was too aggressive**. A 2-second election timeout means any network hiccup lasting more than 2 seconds triggers a full election cycle. Cross-AZ networks routinely see 5-10 ms spikes; this was a 800 ms spike, but with only 2 seconds of tolerance, it was enough.
- **Equal priorities created a tie scenario**. With three data nodes all at priority 1, and an arbiter, the election was a coin flip — except each flip required oplog freshness. In an odd-numbered voter set (3 voting members: 3 data nodes, no arbiter), a secondary can win with 2 votes. But with the arbiter making 4 total members and 3 voters, it should have been fine. The real problem was that the secondaries could not vote for each other due to oplog staleness.
- **No explicit failover priority**. The cluster had been set up with default priorities, meaning no node was designated as the preferred primary. This made elections unpredictable.
- **The bulk write was not rate-limited**. A bulk import generating 500 MB of oplog entries in seconds created a replication backlog that, under degraded network conditions, took 10 minutes to drain.
- **No chaos testing for elections**. The team had never tested what happens when the network degrades but does not fail completely. Partial degradation is harder to detect and more insidious than a full partition.

### Key Principles

1. **electionTimeoutMillis is a safety net, not a performance knob**. Setting it too low trades stability for speed. Always use 10000 ms or higher, especially in cross-AZ deployments.
2. **Priority asymmetry prevents ties**. Assign different priorities to your nodes (e.g., 5, 3, 1) so the cluster always converges on the same preferred primary.
3. **Replication lag kills elections**. A secondary with oplog lag cannot win an election. Monitor lag as a first-class alert.
4. **Test your elections**. Use chaos engineering to simulate network degradation. A healthy cluster should survive a 500 ms RTT spike without losing its primary.
5. **Know your arbiter's limits**. An arbiter helps with quorum but cannot become primary and cannot help when the issue is oplog staleness.

---

## Summary

### Key Configurations: Before vs. After

| Parameter | Before (Problem) | After (Fix) |
|-----------|------------------|-------------|
| `electionTimeoutMillis` | 2000 ms | 10000 ms |
| `heartbeatIntervalMillis` | 2000 ms | 2000 ms (unchanged) |
| `catchUpTimeoutMillis` | 3000 ms | 10000 ms |
| Node A priority | 1 | 3 |
| Node B priority | 1 | 2 |
| Node C priority | 1 | 1 |
| Arbiter priority | 0 | 0 |
| Write concern | w: "majority" | w: "majority" (unchanged) |

### Timeline Summary

```
14:03:12  Network latency spike (2 ms -> 800 ms RTT)
14:03:15  Primary heartbeat timeout from both secondaries
14:03:16  Primary steps down
14:03:17  Election fails — stale oplogs on both secondaries
14:03:30  Election retry fails again
    ...   (33 failed election rounds over 11 minutes)
14:13:45  Secondary B catches up, wins election
14:14:00  Writes resume
```

### Final Checklist

If you are deploying a MongoDB replica set today, verify these items:

- [ ] `electionTimeoutMillis` is at least 10000 ms
- [ ] Priority values are distinct across data nodes (e.g., 5, 3, 1)
- [ ] Alert on secondary lag > 30 seconds
- [ ] Alert on no primary > 30 seconds
- [ ] `w: "majority"` is enforced at the driver level
- [ ] Bulk write operations are split into batches with rate limiting
- [ ] Network RTT between replica set members is < 10 ms under normal conditions
- [ ] Election failover has been tested (chaos engineering)
- [ ] `rs.status()` shows correct voting configuration
- [ ] Oplog size is sufficient for expected bulk operations

---

*Last updated: 2026-06-11*
