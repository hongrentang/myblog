---
title: "MongoDB 复制集选举故障 — 主节点降级后无人胜选，集群 11 分钟无主"
date: 2026-06-11
weight: 100510
slug: "mongodb-replica-election-failure"
tags: ["mongodb", "database", "troubleshooting", "replication", "election"]
categories: ["Troubleshooting"]
description: "一次 MongoDB 复制集选举故障纪实 — 网络延迟飙升导致主节点误降级，因投票数不足和从节点 oplog 落后，选举反复失败，集群 11 分钟无主节点"
keywords: "MongoDB 复制集选举, MongoDB 无主节点, MongoDB 选举超时, MongoDB 复制延迟, MongoDB rs.status, 复制集故障排查"
draft: false
featured: true
cover:
  image: ""
  caption: "MongoDB 复制集选举故障排查"
---

# MongoDB 复制集选举故障 — 主节点降级后无人胜选，集群 11 分钟无主

## 常见搜索关键词 / Common Search Queries

| 中文 | English |
|------|---------|
| MongoDB 复制集选举失败 | MongoDB replica set election failed |
| MongoDB 选举后无主节点 | MongoDB no primary after election |
| MongoDB electionTimeoutMillis 设置过低 | MongoDB electionTimeoutMillis too low |
| MongoDB 仲裁节点不投票 | MongoDB arbiter not voting in election |
| MongoDB 心跳失败导致主节点降级 | MongoDB heartbeat failed step down |
| MongoDB rs.status() 无 PRIMARY | MongoDB rs.status() no PRIMARY |
| MongoDB 从节点 oplog 落后选举失败 | MongoDB secondary stale oplog election |
| MongoDB 选举失败强制重新配置 | MongoDB force reconfig after election failure |
| MongoDB 复制延迟导致选举失败 | MongoDB replication lag causes election failure |
| MongoDB 跨可用区复制集选举调优 | MongoDB cross-AZ replica set election tuning |

---

## 故障经过 / The Incident

### 环境信息 / Environment

| 组件 | 规格 |
|------|------|
| MongoDB 版本 | 7.0.14 (Community) |
| 复制集节点数 | 3 节点（主 + 从 + 从）|
| 仲裁节点 | 1 个仲裁节点（仅投票，不存数据）|
| 写关注 | w: "majority" |
| 读偏好 | primary（默认）|
| 部署方式 | 单区域，3 个可用区 |
| 网络 RTT（正常） | ~1-2 ms |
| 存储引擎 | WiredTiger |
| Oplog 大小 | 10 GB（7.0 默认值）|

### 时间线 / Timeline

| 时间 (UTC) | 事件 |
|------------|------|
| 14:03:00 | 网络维护窗口开始 |
| 14:03:12 | 延迟飙升：节点间 RTT 从 2 ms 升至 800 ms |
| 14:03:15 | 主节点检测到两个从节点心跳超时 |
| 14:03:16 | 主节点降级，转为 SECONDARY 状态 |
| 14:03:17 | 选举触发 — 两个从节点均参选 |
| 14:03:19 | 第 1 轮选举失败 — 两个从节点 oplog 均落后 |
| 14:03:30 | 第 2 轮选举失败 — 同样原因，超时 |
| 14:04:00 - 14:13:30 | 持续重试选举 — 全部失败 |
| 14:13:45 | 从节点 B 追平 oplog，赢得选举，成为 PRIMARY |
| 14:14:00 | 应用重新连接，写入恢复 |
| 14:14:30 | 原主节点重新加入成为 SECONDARY |

**故障时长：11 分钟** — 期间集群所有写入均失败。

### 症状 / Symptoms

- 应用团队报告所有写入端点返回 **HTTP 500 错误**
- MongoDB 驱动日志反复出现 `NotWritablePrimary` / `not primary` 错误
- `rs.status()` 显示所有三个数据节点均为 `stateStr: "SECONDARY"`
- 仲裁节点显示 `stateStr: "ARBITER"`，但无法单方面打破平局
- `rs.isMaster()` 所有节点均返回 `"ismaster": false`
- MongoDB 日志充满 `Election failed, sleeping` 和 `heartbeat failed` 消息
- Alertmanager 触发 `MongoDB_NoPrimary` 告警（维护窗口期间被静默）

---

## 背景 / Background

### MongoDB 复制架构

MongoDB 复制集使用 **oplog**（操作日志）来复制数据。主节点上的每次写入都会记录在主节点的 oplog 中。从节点异步拉取这些操作并应用到自己的数据文件中。这类似于 MySQL 的二进制日志复制，但在文档级别操作。

关键概念：

- **Oplog**：一个 capped 集合（`local.oplog.rs`），存储所有写入操作。每个从节点通过从主节点（或其他从节点）回放操作来维护自己的 oplog。
- **复制延迟**：从节点最后应用的操作与主节点最后操作之间的时间差。通过比较 `rs.status()` 中的 `optimeDate` 值来测量。
- **心跳**：每个成员默认每 2 秒（`heartbeatIntervalMillis: 2000`）向其他成员发送心跳。心跳携带状态信息，包括成员的当前 term 和 optime。
- **Term**：一个单调递增的数字，代表一个领导任期。每次选举都会增加 term。

### 选举过程

当复制集需要选择新主节点时（当前主节点降级、不可达或更高优先级的成员加入），会发生以下过程：

1. **投票请求**：任何检测到无法联系主节点的从节点通过向所有其他成员发送 `voteRequest` 来发起选举。
2. **投票响应**：每个成员投票给它认为最合适的候选者。合适性由以下因素决定：
   - **优先级**：优先级更高的成员更受青睐。
   - **Oplog 新鲜度**：oplog 落后于投票者自身 oplog 的从节点将不会获得投票。
   - **Term**：每个 term 每个成员只投票一次。
3. **法定人数**：候选者需要获得多数票（ceil(N/2)）才能成为主节点。对于 3 节点 + 仲裁节点的设置，需要 3 个投票成员中的 2 票。
4. **胜选**：如果没有候选者达到多数，选举失败并在随机延迟后重试。

### 选举时间配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `electionTimeoutMillis` | 10000 (10s) | 从节点检测到主节点故障前等待的最大时间 |
| `heartbeatIntervalMillis` | 2000 (2s) | 心跳间隔 |
| `heartbeatTimeoutSecs` | 10 | 心跳被认为失败的秒数 |

当 `electionTimeoutMillis` 过期仍未收到主节点响应时，从节点假定主节点已宕机并发起选举。如果此值设置过低，集群会变得**不稳定** — 短暂的网络波动会触发不必要的选举。

### 仲裁节点的角色

仲裁节点是一个轻量级成员，**参与选举但不持有数据**。它的存在仅仅是为了在偶数节点数的复制集中提供打破平局的一票。关键规则：

- 仲裁节点可以投票但**不能成为主节点**
- 仲裁节点不复制数据，也没有 oplog
- 仲裁节点通常部署在独立的低资源主机上
- 仲裁节点的投票计入多数阈值

---

## 排查过程 / Investigation

以下是故障期间排查集群状态所执行的步骤。

### 1. 检查复制集状态

```javascript
rs.status()
```

输出显示所有三个数据节点都处于 `SECONDARY` 状态：

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

集群中没有 PRIMARY。仲裁节点在线，但由于两个从节点优先级相同，无法打破平局。

### 2. 检查当前主节点

```javascript
rs.isMaster()
```

所有节点均返回 `"ismaster": false`。确认集群中不存在可写节点。

### 3. 检查选举次数

```javascript
rs.status().electionParticipantMetrics
```

显示了选举尝试次数。在我们的案例中，11 分钟内尝试了超过 30 轮选举。

### 4. 检查复制延迟和 Oplog 窗口

```javascript
rs.printSecondaryReplicationInfo()
```

两个从节点的输出均显示显著延迟：

```
source: node-b:27017
    syncedTo: '2026-06-11T14:03:15.123Z'
    lag: 45s (estimated)
source: node-c:27017
    syncedTo: '2026-06-11T14:03:20.456Z'
    lag: 40s (estimated)
```

降级发生时，从节点落后原主节点 40-45 秒。这种延迟是由一次**批量写入操作**（批量导入约 50 万文档）引起的，该操作在网络尖峰到来时仍在进行。

检查 oplog 窗口：

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

Oplog 窗口健康（约 2 小时），oplog 未被截断。

### 5. 检查 mongod 日志中的选举记录

```bash
grep -i "election\|stepdown\|term" /var/log/mongodb/mongod.log | tail -30
```

关键日志行：

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

日志显示一个重复模式：每个从节点都拒绝为另一个投票，因为每个都认为自己的 oplog 更领先。两者都无法获得多数票。

### 6. 检查心跳历史

```bash
grep -i "heartbeat" /var/log/mongodb/mongod.log | tail -20
```

```
2026-06-11T14:03:12.789Z I REPL [replexec-0] Heartbeat failed after 10 retries to node-b:27017; RS102: heartbeat timeout
2026-06-11T14:03:12.790Z I REPL [replexec-0] Heartbeat failed after 10 retries to node-c:27017; RS102: heartbeat timeout
2026-06-11T14:03:13.001Z I REPL [replexec-0] Heartbeat failed again to node-b:27017; RS102
2026-06-11T14:03:13.002Z I REPL [replexec-0] Heartbeat failed again to node-c:27017; RS102
```

800 ms 的 RTT 延迟导致心跳响应远远超过 2 秒的间隔窗口，触发了超时。

### 7. 检查投票配置

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

所有三个数据节点优先级相同（1）。这是默认配置，但在 3+1 拓扑中，相同优先级意味着选举中没有成员拥有优势。仲裁节点优先级为 0（不能成为主节点），但投票权计入。

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

`electionTimeoutMillis` 设置为 **2000 ms**（允许的最小值）。这是一个非常激进的设置。默认值是 10000 ms。团队为了在计划维护期间加速故障转移而降低了此值，但副作用是即使微小的网络抖动也会触发选举。

### 8. 检查节点间网络延迟

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

RTT 约 800 ms — 没有丢包但延迟极高。这证实了网络问题（后来追踪到维护窗口期间一台交换机的配置错误）。

### 9. 检查 serverStatus 指标

```javascript
db.adminCommand({serverStatus:1}).replication
```

每个从节点的 `serverStatus` 输出显示：

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

超过 33 次失败的选举尝试，仅 1 次胜选（最终恢复集群的那次）。

---

## 根因 / Root Cause

根因链涉及三个相互交织的故障：

### 触发因素：网络延迟飙升

在计划中的网络维护窗口期间，一次交换机配置错误导致复制集成员之间的 RTT 从约 2 ms 飙升到约 800 ms。这相当于跨洲链路的延迟水平，而非本地网络。高延迟导致心跳请求超过超时阈值。

### 故障 1：主节点降级

主节点（node-a）检测到两个从节点（node-b、node-c）的心跳失败。由于默认的 `heartbeatTimeoutSecs` 为 10 秒，`heartbeatIntervalMillis` 为 2000 ms，主节点将两个从节点标记为不可达并自行降级。这是一个**合理的自我保护**行为 — 无法与多数投票成员通信的主节点应当降级以防止脑裂。

### 故障 2：选举死锁

两个从节点由于正在进行批量写入操作（批量导入约 50 万文档）而存在 **40-45 秒的复制延迟**。当选举开始时，每个从节点都检查对方的 oplog 新鲜度。Node-b 的 oplog 在时间戳 T1；node-c 的 oplog 在时间戳 T2（略落后）。两者都无法为对方投票，因为：

- Node-b 不会投票给 node-c，因为 node-c 的 oplog 落后于 node-b。
- Node-c 不会投票给 node-b，因为从 node-c 的视角看，node-b 的 optime 处于**不同的 term**（由于降级和选举循环）。

仲裁节点投票给先来请求的成员，但候选者仍然需要 **2 票**（3 个投票成员中的多数）— 且只能获得 1 票（自身 + 仲裁节点，但仲裁节点的一票只算 1 票，还需要另 1 票）。

### 故障 3：激进的 electionTimeoutMillis

```javascript
settings.electionTimeoutMillis = 2000
```

团队将此值设置为最小值（2000 ms），以为这样可以加速故障转移。 unintended consequence：

- 当选举失败时，下一个从节点只等待 2 秒就发起新一轮选举
- 追赶阶段（`catchUpTimeoutMillis` 为 3000 ms）对于从节点在 800 ms 延迟下获取积压的 oplog 条目来说太短
- 每次选举递增的 term 使得落后的从节点更难参与

### 为什么持续 11 分钟？

每轮选举大约需要 2-3 秒（选举超时 + 投票交换 + 故障检测）。33 轮失败的累积时间：

- 每轮约 3 秒 x 33 轮 = 约 99 秒的选举开销
- 加上追赶尝试的时间（总计约 50 秒）
- 加上重试之间的退避（指数增长：1 秒、2 秒、4 秒、8 秒...）
- 真正的瓶颈：从节点需要在 800 ms 延迟下清除约 600 秒（10 分钟）的复制积压

批量写入产生了约 500 MB 的 oplog 条目。在 800 ms RTT 下，oplog 同步的持续吞吐量降至约 1-2 MB/s，而正常情况约为 50 MB/s。清除积压正好花了 10 分钟。

### 故障链示意图

```
  网络延迟飙升 (2ms -> 800ms)
           |
    心跳超时
           |
  主节点降级
           |
  选举触发
    /            \
   /              \
Node-b 有延迟   Node-c 有延迟
   \              /
    \            /
  两者都无法获得多数票
           |
   electionTimeoutMillis=2s
           |
  过早超时，重试循环
           |
  11 分钟无主节点
```

---

## 解决方案 / Resolution

### 紧急恢复（故障期间）

当集群没有主节点时，有两个选项。

**选项 1：强制重新配置 — 优先指定某个从节点**

```javascript
cfg = rs.conf()
cfg.members[1].priority = 10   // node-b 获得最高优先级
cfg.members[2].priority = 1    // node-c 较低优先级
cfg.members[3].priority = 0    // 仲裁节点保持 0
rs.reconfig(cfg, {force: true})
```

`{force: true}` 标志允许在多数节点不健康时重新配置。这强制集群以 node-b 为新主节点收敛。Node-b 成为主节点后会追赶其他节点。

**选项 2：手动提升从节点**

在已降级的原主节点上：

```javascript
rs.stepDown(300)
```

这告诉当前节点（处于 SECONDARY 状态）在 300 秒内不参选，给另一个节点胜选的机会。但在我们的案例中这不起作用，因为问题不是旧主节点阻塞选举 — 而是没有一个从节点能获得足够的票数。

**选项 3：以更高优先级配置重启 mongod**

如果无法执行 reconfig（例如网络分区阻止命令到达所有节点），可以在某个从节点上重启 mongod 进程，使用修改后的配置文件设置 `priority: 10` 来强制其成为主节点。

**实际生效的方案**

在这次故障中，解决方案是**时间**。大约 10 分钟后，node-b 完成了对 oplog 缓冲中批量写入操作的应用。一旦其 `optime` 追赶上，它就获得了 node-c 和仲裁节点的投票，以 2 票赢得选举，成为主节点。

随后团队应用了以下配置修复以防止再次发生。

### 即时配置修复

```javascript
cfg = rs.conf()
cfg.settings.electionTimeoutMillis = 10000   // 从 2000 ms 改为 10000 ms
cfg.members[0].priority = 3                  // node-a：首选主节点
cfg.members[1].priority = 2                  // node-b：第一故障转移目标
cfg.members[2].priority = 1                  // node-c：第二故障转移目标
cfg.members[3].priority = 0                  // 仲裁节点保持仲裁者角色
rs.reconfig(cfg)
```

重新配置后：

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

### 长期改进措施

| 类别 | 措施 | 详情 |
|------|------|------|
| 配置 | 增大 `electionTimeoutMillis` | 设置为 10000 ms（默认值），适用于跨可用区部署。网络 RTT 超过 10 ms 时设为 15000 ms 也是合理的。 |
| 配置 | 设置明确的优先级 | 使用 5、3、1（或类似）防止平局。在偶数投票节点集中永远不要让所有成员保持 priority 1。 |
| 配置 | 调整 `catchUpTimeoutMillis` | 如果批量操作期间的复制延迟常见，增大到 10000 ms。 |
| 应用 | 固定写关注 | 生产写入始终使用 `w: "majority"`。避免使用 `w: 1`（未确认写入）。 |
| 应用 | 批量写入限流 | 限制批量操作的速率以防止 oplog 积压。使用 `maxTimeMS` 和分批处理。 |
| 监控 | 跟踪选举指标 | 对 `mongodb_rs_state != 1`（非主节点）设置告警。监控 `mongodb_rs_member_optime_date` 以检测延迟。 |
| 监控 | 从节点延迟告警 | 如果任何从节点的延迟超过 30 秒则告警。这是选举故障的金丝雀。 |
| 监控 | 心跳指标 | 绘制 `mongodb_rs_member_heartbeat_latency` 图表以早期检测网络抖动。 |
| 基础设施 | 混沌工程 | 使用 `tc`（流量控制）或服务网格定期执行网络分区测试。 |
| 基础设施 | 考虑 5 节点复制集 | 5 节点集（3 数据 + 2 数据，或 3 数据 + 2 仲裁）提供更高的容错性并避免偶数投票者平局问题。 |
| 驱动 | 驱动层 writeConcern | 在 MongoDB 驱动级别设置 `writeConcern: "majority"`，而不仅限于应用逻辑层面。 |

### 修复后验证

应用修复后，验证集群健康：

```javascript
rs.status().members.forEach(m => {
  print(m.name + ": " + m.stateStr + " | optime: " + m.optimeDate);
})
```

输出应显示一个 PRIMARY、两个 SECONDARY 和一个 ARBITER，且 optime 日期接近。

```bash
# 使用 failpoint 模拟干净的选举测试（仅限开发环境）
mongosh --eval "db.adminCommand({replSetTest: {name: 'rs0', forceElection: true}})"
```

---

## 经验教训 / Lessons Learned

### 做得好的方面

- 仲裁节点在整个故障期间保持可达，防止了集群完全不可用（集群有法定人数，只是没有候选者能达到多数）。
- 应用层面的重试和熔断器防止了级联故障传播到下游服务。
- 维护窗口安排在低流量时段，因此影响范围有限。
- MongoDB 的选举重试机制最终生效 — 只是花费的时间远超预期。

### 做得不好的方面

- **electionTimeoutMillis = 2000 ms 过于激进**。2 秒的选举超时意味着任何持续超过 2 秒的网络波动都会触发完整的选举周期。跨可用区网络通常会看到 5-10 ms 的尖峰；这次是 800 ms 的尖峰，但仅有 2 秒的容差，已经足够了。
- **相同优先级造成平局**。三个数据节点均为 priority 1，加上仲裁节点，选举像抛硬币 — 但每次抛硬币都需要 oplog 新鲜度。在奇数投票节点集中（3 个投票成员：3 个数据节点，无仲裁），从节点可以以 2 票获胜。但加上仲裁节点共 4 个成员和 3 个投票者，本应没问题。真正的问题是从节点由于 oplog 过时而无法相互投票。
- **没有明确的故障转移优先级**。集群使用默认优先级设置，意味着没有节点被指定为首选主节点。这使得选举不可预测。
- **批量写入没有限流**。批量导入在几秒内生成了 500 MB 的 oplog 条目，在网络降级条件下需要 10 分钟才能清除的复制积压。
- **没有进行选举的混沌测试**。团队从未测试过网络降级但不完全中断的情况。部分降级比完全分区更难检测且更具隐蔽性。

### 关键原则

1. **electionTimeoutMillis 是安全网，不是性能旋钮**。设置过低是用稳定性换速度。始终使用 10000 ms 或更高，特别是在跨可用区部署中。
2. **优先级不对称防止平局**。为你的节点分配不同的优先级（例如 5、3、1），使集群始终收敛到相同的首选主节点。
3. **复制延迟会杀死选举**。oplog 落后的从节点无法赢得选举。将延迟作为一等告警指标进行监控。
4. **测试你的选举机制**。使用混沌工程模拟网络降级。健康的集群应该能承受 500 ms 的 RTT 尖峰而不失去主节点。
5. **了解仲裁节点的限制**。仲裁节点有助于法定人数，但不能成为主节点，也无法在 oplog 过时问题上提供帮助。

---

## 总结 / Summary

### 关键配置：修复前后对比

| 参数 | 修复前（问题） | 修复后 |
|------|---------------|--------|
| `electionTimeoutMillis` | 2000 ms | 10000 ms |
| `heartbeatIntervalMillis` | 2000 ms | 2000 ms（未更改）|
| `catchUpTimeoutMillis` | 3000 ms | 10000 ms |
| Node A 优先级 | 1 | 3 |
| Node B 优先级 | 1 | 2 |
| Node C 优先级 | 1 | 1 |
| 仲裁节点优先级 | 0 | 0 |
| 写关注 | w: "majority" | w: "majority"（未更改）|

### 时间线总结

```
14:03:12  网络延迟飙升（2 ms -> 800 ms RTT）
14:03:15  主节点心跳超时（两个从节点均超时）
14:03:16  主节点降级
14:03:17  选举失败 — 两个从节点 oplog 均落后
14:03:30  选举重试再次失败
    ...   （11 分钟内 33 轮选举失败）
14:13:45  从节点 B 追平，赢得选举
14:14:00  写入恢复
```

### 最终检查清单

如果你今天要部署 MongoDB 复制集，请验证以下项目：

- [ ] `electionTimeoutMillis` 至少为 10000 ms
- [ ] 数据节点的优先级值各不相同（例如 5、3、1）
- [ ] 从节点延迟超过 30 秒时告警
- [ ] 无主节点超过 30 秒时告警
- [ ] 驱动层面强制使用 `w: "majority"`
- [ ] 批量写入操作已分批处理并限流
- [ ] 正常情况下复制集成员间的网络 RTT 小于 10 ms
- [ ] 选举故障转移已进行过测试（混沌工程）
- [ ] `rs.status()` 显示正确的投票配置
- [ ] Oplog 大小足以容纳预期的批量操作

---

*最后更新: 2026-06-11*
