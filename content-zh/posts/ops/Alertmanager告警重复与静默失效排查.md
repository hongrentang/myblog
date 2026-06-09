---
title: "Alertmanager 告警重复与静默失效排查——凌晨三点 PagerDuty 疯狂报警事件"
date: 2026-06-09
weight: 100460
slug: "alertmanager-duplicate-alerts-silences"
tags: ["prometheus", "alertmanager", "monitoring", "troubleshooting", "alerting"]
categories: ["Troubleshooting"]
description: "Alertmanager 生产事故实录——group_by 和 group_wait 配置不当导致每分钟 2000 条重复告警，配置重载导致静默规则失效的完整排查过程"
keywords: "alertmanager 告警重复, prometheus alertmanager 静默规则失效, alertmanager group_by 配置, alertmanager 分组, pagerduty 告警风暴"
draft: false
featured: true
cover:
  image: ""
  caption: "Alertmanager——排障实战"
---

# Alertmanager 告警重复与静默失效排查——凌晨三点 PagerDuty 疯狂报警事件

## 常见搜索词

- alertmanager 告警重复 解决
- prometheus alertmanager 静默规则不生效
- alertmanager group_by 配置错误
- alertmanager 重载配置后静默丢失
- alertmanager 集群脑裂 通知重复
- pagerduty 告警风暴 alertmanager
- amtool silence list 不匹配
- alertmanager group_wait 每个告警单独发送
- prometheus alertmanager 重复 pagerduty 告警
- alertmanager 集群 gossip 失败

---

## 故障经过

### 环境信息

| 组件 | 版本/规格 |
|------|----------|
| Prometheus | 2.45 |
| Alertmanager | 0.27 |
| 通知渠道 | PagerDuty（主）、Slack（辅） |
| 集群规模 | 50 个 Kubernetes 节点，3 个 Alertmanager 副本 |
| 告警规则 | 2000+ 条记录与告警规则 |

### 时间线

**02:45** — 运维团队对 Kubernetes `DaemonSet` 执行滚动更新，涉及全部 50 个节点。每个节点上的 Pod 依次重启，触发了大量 `KubePodCrashLooping`、`PodNotReady` 和 `NodeCondition` 告警。当晚值班工程师此前已为计划内维护创建了静默规则，本应能抑制部分告警噪音。

**02:47** — PagerDuty 开始疯狂报警。工程师手机每分钟收到 50+ 条推送通知。Slack `#alerts` 频道以每分钟 30 条消息的速度被刷屏。PagerDuty 移动端因通知过载崩溃了两次。

**02:52** — 工程师确认了 PagerDuty 事件并查看 Alertmanager 状态页面，期望看到已分组合并的告警。但展现在眼前的是 200+ 个独立的告警组，每个组只有一条告警。

**02:55** — 工程师尝试用 `amtool silence add` 静默正在泛滥的告警，却发现几小时前创建的静默规则已经完全失效。`amtool silence list` 显示静默规则状态为 active，但它们对告警没有任何抑制效果。

**03:00** — 混乱持续 15 分钟后，工程师已收到超过 200 条 PagerDuty 推送通知和 500+ 条 Slack 消息。生产事件升级至 SRE 主管。

### 症状

- **PagerDuty 告警风暴**：每分钟 50+ 条通知，移动端崩溃两次
- **静默规则失效**：之前正常工作的维护静默规则完全不起作用
- **告警未分组**：每个 Pod 重启都触发一条独立告警通知
- **Slack 刷屏**：`#alerts` 频道 15 分钟内涌入 500+ 条消息
- **值班疲劳**：工程师收到 200+ 个未接来电和 500+ 条 Slack 告警

---

## 背景

要理解问题根源，需要先了解 Alertmanager 处理告警的核心机制。

### Alertmanager 架构

Alertmanager 是 Prometheus 监控栈中的路由与通知层。它从 Prometheus 接收告警后，经过三个阶段处理：

1. **抑制（Inhibition）** — 当某些告警处于活跃状态时，抑制其他相关告警（例如在 `NodeDown` 时抑制 `PodCrashLooping`）。
2. **分组（Grouping）** — 根据标签匹配将相似告警合并为一条通知。
3. **静默（Silencing）** — 基于用户定义的标签匹配器抑制特定告警的通知。

### 配置 vs 运行时状态

Alertmanager 中一个关键的区别在于配置与运行时状态：

- **配置**（`alertmanager.yml`）：定义路由、接收器、分组规则、抑制规则和通知模板。在启动或重载时从磁盘加载。
- **运行时状态**：活跃的静默规则、通知历史记录和集群成员信息。通过 `amtool` 或 API 创建的静默规则**仅存在于运行时状态中**——它们**不属于** `alertmanager.yml`。

当 Alertmanager 重载配置时（通过 `kill -HUP` 或 `POST /-/reload`），它会重新从磁盘读取 `alertmanager.yml`，但**保留**内存中的静默规则——**除非**重载时指定了新的静默文件路径或数据目录丢失。这是很多团队踩坑的地方。

### 集群模式与 Gossip 协议

Alertmanager 支持多副本高可用模式。副本之间通过 **Gossip 协议**（基于 Protobuf over TCP，默认端口 9094）通信，用于：

- 在所有实例间同步活跃静默规则
- 去重通知投递——每个告警组只应有一个副本发送通知
- 共享通知历史，防止主节点切换后重复发送

Gossip 要求所有对等节点之间具有稳定的网络连通性。如果有节点不可达，就会出现脑裂，每个分区都会独立触发通知。

---

## 排查过程

值班工程师与 SRE 主管按以下步骤进行了系统性排查：

### 第一步：检查 Alertmanager 状态界面

```bash
# 查询 Alertmanager 状态端点
curl -s http://localhost:9093/api/v2/status | jq .
```

响应内容：

```json
{
  "config": { ... },
  "cluster": {
    "status": "ready",
    "peers": [
      {"name": "pod-0", "address": "10.0.1.10:9094"},
      {"name": "pod-1", "address": "10.0.1.11:9094"},
      {"name": "pod-2", "address": "10.0.1.12:9094"}
    ]
  },
  "uptime": "2026-06-08T14:00:00Z",
  "versionInfo": {"version": "0.27.0"}
}
```

三个对等节点都可见，但集群 gossip 统计显示存在连接失败。

### 第二步：查看活跃告警

```bash
amtool alert list
```

输出显示 1200+ 条活跃告警——每条都有独特的标签组合。滚动更新期间出现大量告警是预期的，但分组机制应该能将其合并才对。

### 第三步：检查静默规则

```bash
amtool silence list
```

静默规则存在，且状态为 `active`：

```
ID         Matchers                          Status   Ends
abc123     alertname="NodeNotReady"          Active   2026-06-09 06:00:00
def456     alertname=".*" pod="web-.*"       Active   2026-06-09 06:00:00
```

然而匹配这些模式的告警仍然在触发通知。这是第一个危险信号——静默规则**显示活跃但实际不匹配**。

### 第四步：对比静默匹配器与告警标签

```bash
# 查看某条告警的标签
amtool alert query alertname="NodeNotReady" --json | jq '.[0].labels'
```

```json
{
  "alertname": "NodeNotReady",
  "instance": "node-23",
  "namespace": "production",
  "node_name": "node-23",
  "pod": "",
  "severity": "critical"
}
```

对比静默匹配器：

```
Silence: alertname="NodeNotReady"
```

这条应该匹配。但 SRE 主管注意到一个细微问题——第二条静默规则使用了 `pod="web-.*"`，但告警中的标签名已经变了。在最近一次配置变更中，告警规则的标签 `pod` 被改名为 `pod_name`，但静默规则仍然在匹配旧的 `pod` 标签。

此外，部分静默规则创建时使用了 `regex: true`，但标签变更后正则表达式在语法上已经不匹配告警标签了，导致匹配失败。

### 第五步：检查分组配置

```bash
amtool config show
```

`route` 段揭示了重复通知的根因：

```yaml
route:
  receiver: "pagerduty-critical"
  group_by: ['...']
  group_wait: 5s
  group_interval: 5m
  repeat_interval: 4h
```

**`group_by: ['...']`** 是一个陷阱。这里的三个点（省略号）告诉 Alertmanager 按照**所有**标签分组。这意味着每个独特的标签值组合都会创建一个新的告警组。面对 50 个节点和 10 种不同的告警类型，这会产生 500+ 个独立的告警组，每个组都触发自己的通知。

`group_wait: 5s` 让情况更糟——它只给 5 秒钟等待更多告警到来就立即发送通知。滚动更新过程中新告警持续产生，5 秒远远不足以有效批量合并。

### 第六步：查看 Alertmanager 日志

```bash
kubectl logs -n monitoring alertmanager-0 | grep -i error
```

日志显示：

```
level=error msg="silence abc123: failed to match" error="regex compile error"
level=warn msg="cluster gossip: 2 of 3 peers unreachable" count=2
level=error msg="notification failed: rate limit exceeded" receiver=pagerduty
```

确认了三个问题：
1. 静默规则正则表达式编译失败
2. 集群 Gossip 问题——脑裂
3. 告警风暴触发了 PagerDuty 速率限制

### 第七步：检查 Gossip / 集群状态

```bash
curl -s http://localhost:9093/api/v2/status | jq '.cluster'
```

```json
{
  "status": "ready",
  "peers": [
    {"name": "pod-0", "address": "10.0.1.10:9094"},
    {"name": "pod-1", "address": "10.0.1.11:9094"},
    {"name": "pod-2", "address": "10.0.1.12:9094"}
  ],
  "failedPeers": [
    {"name": "pod-1", "address": "10.0.1.11:9094"},
    {"name": "pod-2", "address": "10.0.1.12:9094"}
  ]
}
```

Pod-0 被隔离了——pod-1 和 pod-2 在 gossip 端口上不可达。检查网络策略发现，最近的一次 `NetworkPolicy` 更新意外阻止了部分 Pod 之间的 9094 端口通信。

3 个对等节点中有 2 个不可达，Alertmanager 集群实际上处于脑裂状态。Pod-0 在发送自己的通知，pod-1 和 pod-2 各自也在发送通知——通知量翻了三倍。

### 第八步：检查通知指标

```bash
# PromQL 查询
rate(alertmanager_notifications_total{receiver="pagerduty"}[5m])
```

查询结果显示每分钟 **2000 条通知**——远超 PagerDuty 每分钟 120 条通知的速率限制。PagerDuty 的限流反过来导致背压，通知发送失败后不断重试，产生了更多噪音。

---

## 根因

五个因素叠加导致了这次事故：

### 1. `group_by: ['...']`——全局分组

**`group_by: ['...']`**（三个点）等同于按**每一个**标签分组。在生产环境中几乎永远不应该使用：

- 每个独特的标签值组合都创建一个独立的告警组
- 50 个节点上 10 种告警类型进行滚动更新 = 500+ 个组
- 每个组都向 PagerDuty 发送自己的通知

正确的配置是指定明确的、基数受限的标签：

```yaml
# 错误——每个独特标签组合都创建独立分组
group_by: ['...']

# 正确——按告警名称和严重级别分组
group_by: ['alertname', 'severity']
```

### 2. `group_wait: 5s`——批处理窗口不足

`group_wait` 控制 Alertmanager 在发送组内第一条通知之前等待更多告警的时间。只有 5 秒的情况下：

- 同一节点上相隔 6 秒到达的告警会作为独立通知发送
- 滚动更新期间，告警批次在数分钟内陆续到达，而非数秒
- 30 秒到 1 分钟的 `group_wait` 本可以将大量告警合并为一条通知

### 3. 配置重载导致静默规则丢失

这是最隐蔽的问题。当晚早些时候，团队对 Alertmanager 执行了配置变更：

```bash
kill -HUP $(pidof alertmanager)
# 或者
curl -X POST http://localhost:9093/-/reload
```

正常情况下，Alertmanager 重载会保留内存中的静默规则。但这次重载时，**启动命令已被修改**，指向了一个新的数据目录（`--data.dir`），导致 Alertmanager 以空的静默存储初始化。所有通过 API 和 `amtool` 创建的活跃静默规则都变成了**孤儿**——它们仍然出现在 `amtool silence list` 中（来自旧存储），但已经是无法匹配的过期引用。

静默规则显示 `status: active`，但它们的内部状态已经损坏。Alertmanager 无法正确评估它们是否匹配传入的告警。

### 4. 标签重命名导致静默正则不匹配

在之前的一次 Prometheus relabeling 配置变更中，告警标签的 `pod` 被改名为 `pod_name`。静默匹配器仍然引用了旧的 `pod` 标签：

```
# 旧静默规则——不再匹配
pod="web-.*"  (regex: true)

# 标签变更后告警上的新标签
pod_name="web-frontend-abc123"
```

Alertmanager 的静默匹配是严格的——如果告警上不存在该标签，匹配器根本不会触发。静默规则显示为活跃但实际完全无效。

### 5. 集群脑裂——Gossip 通信失败

一次 `NetworkPolicy` 更新阻止了 Alertmanager Pod 之间的 TCP 9094 端口通信，导致 Gossip 同步中断：

| Pod | 状态 | 影响 |
|-----|------|------|
| pod-0 | 被隔离，无法联系 pod-1 和 pod-2 | 发送自己的通知 |
| pod-1 | 无法联系 pod-0，可联系 pod-2 | 发送自己的通知，与 pod-0 重复 |
| pod-2 | 无法联系 pod-0，可联系 pod-1 | 发送自己的通知，与 pod-0 重复 |

本应每个告警组一条通知，集群却发送了**三条**——每个分区各一条。通知量翻了三倍，直接导致 PagerDuty 速率限制被突破。

---

## 解决方案

### 应急响应

**1. 清除失效静默规则并重新创建：**

```bash
# 清除所有失效的静默规则
amtool silence expire --all

# 使用正确的匹配器重新创建静默规则
amtool silence add \
  alertname="NodeNotReady" \
  matcher="severity=critical" \
  --duration=4h \
  --comment="维护窗口 - 滚动更新"

# 添加使用正确标签名（pod_name 而非 pod）的静默规则
amtool silence add \
  alertname=".*" \
  matcher="pod_name=web-.*" \
  matcher="severity=critical" \
  --duration=4h \
  --comment="Web 服务滚动更新"

# 验证静默规则是否匹配
amtool silence add --verify \
  alertname="KubePodCrashLooping" \
  matcher="severity=warning" \
  --duration=1h
```

**2. 修复分组配置：**

```bash
# 编辑 alertmanager.yml
---
route:
  receiver: "pagerduty-critical"
  group_by: ['alertname', 'namespace', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**3. 使用一致的数据目录重载：**

```bash
# 确保数据目录被保留
alertmanager --config.file=/etc/alertmanager/alertmanager.yml \
  --data.dir=/var/lib/alertmanager \
  --cluster.listen-address=0.0.0.0:9094

# 重载配置
kill -HUP $(pidof alertmanager)
```

**4. 修复集群网络：**

```bash
# 更新 NetworkPolicy 放行 9094 端口的 Gossip 流量
# 修复后验证集群健康状态
curl -s http://localhost:9093/api/v2/status | jq '.cluster.failedPeers'
# 预期：[]（空数组）
```

**5. 暂停 PagerDuty 自动确认：**

作为最后手段，PagerDuty 集成被临时禁用 10 分钟以阻止通知洪流，修复完成后再重新启用。

### 验证修复

应用变更后：

```bash
# 验证无失败对等节点
curl -s http://localhost:9093/api/v2/status | jq '.cluster.failedPeers'
# → []

# 验证通知量下降
# PromQL: rate(alertmanager_notifications_total[5m])
# 修复前：2000/min
# 修复后：~5/min（合并为有意义的批次）

# 验证静默规则匹配
amtool silence list
# 显示新的静默规则正在有效抑制
```

### 长期改进

**1. 静默规则基础设施即代码：**

将 Alertmanager 静默规则存储在 Terraform 或 Helm values 中：

```terraform
resource "alertmanager_silence" "maintenance" {
  matchers {
    name    = "alertname"
    value   = "NodeNotReady"
    is_regex = false
  }
  matchers {
    name    = "severity"
    value   = "critical"
    is_regex = false
  }
  duration = "4h"
  comment  = "维护窗口 - 滚动更新"
}
```

或者对于 Helm 部署的 Alertmanager，使用 `extraSilences`：

```yaml
# values.yaml
alertmanager:
  extraSilences:
    - matchers:
        - name: alertname
          value: "NodeNotReady"
      duration: 4h
      comment: "维护窗口"
```

**2. 显式分组标签——永远不要使用 `group_by: ['...']`：**

```yaml
# 始终指定明确的分组标签
route:
  receiver: "pagerduty-critical"
  group_by: ['alertname', 'namespace', 'severity', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**3. 监控集群健康：**

在 Prometheus 中添加以下告警规则：

```yaml
groups:
  - name: alertmanager-cluster
    rules:
      - alert: AlertmanagerClusterSplitBrain
        expr: |
          count(alertmanager_cluster_members) > 1
          and
          (count(alertmanager_cluster_members)
           - count(alertmanager_cluster_failed_peers) == 1)
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Alertmanager 集群可能处于脑裂状态"

      - alert: AlertmanagerClusterFailedPeers
        expr: |
          alertmanager_cluster_failed_peers > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Alertmanager 集群存在失败对等节点"
```

**4. 应用前验证静默匹配器：**

```bash
# 在创建静默规则前验证是否匹配
amtool silence add --verify \
  alertname="KubePodCrashLooping" \
  matcher="namespace=production" \
  --duration=1h

# 或用现有告警进行测试
amtool silence add --dry-run \
  alertname="KubePodCrashLooping" \
  matcher="namespace=production" \
  --duration=1h
```

**5. Git + CI/CD 管理配置：**

- 将 `alertmanager.yml` 纳入版本控制
- 使用 CI/CD 流水线进行配置变更（ArgoCD、Flux 或自定义脚本）
- 添加自动化验证：`amtool check-config alertmanager.yml`
- 永远不要直接从 shell 执行 `kill -HUP`，必须确认配置正确

**6. PagerDuty 告警疲劳规则：**

创建一条 PagerDuty "告警疲劳"规则，对来自同一 Alertmanager 接收器的通知进行限流（例如每 5 分钟最多 30 条通知），并向运维团队发出警告而非直接静默关键告警。

---

## 经验教训

### 做得好的方面

- 值班工程师熟练掌握了 `amtool` CLI 工具和关键的诊断命令
- SRE 主管快速介入，团队有结构化的排查流程
- Alertmanager 状态 API（`/api/v2/status`）即时提供了集群健康状态的可见性
- Prometheus 指标（`alertmanager_notifications_total`）给出了问题和修复的量化验证

### 做得不好的方面

| 问题 | 类别 | 影响 |
|------|------|------|
| `group_by: ['...']` | 配置 | 500+ 个独立告警组 |
| `group_wait: 5s` | 配置 | 告警无法批量合并 |
| 配置重载导致静默丢失 | 运维 | 已知问题的告警抑制失效 |
| 静默标签不匹配 | 配置 | 正则匹配器静默失败 |
| 集群 Gossip 被阻断 | 网络 | 通知量翻三倍 |
| 缺少集群健康监控 | 可观测性 | 脑裂数小时未被发现 |

### 核心教训

1. **永远不要在生产环境使用 `group_by: ['...']`**——始终指定明确且基数受限的标签。仔细思考哪些标签能产生有用的告警组（例如 `alertname` + `severity` + `namespace`）。

2. **Alertmanager 静默规则是运行时状态**——它们存在于内存中而非配置文件中。做好备份，以代码形式管理，永远不要假设配置重载是无害的。

3. **严格测试静默匹配器**——使用 `amtool silence add --verify` 确认匹配器能匹配真实告警。一个不匹配的静默规则比没有静默规则更可怕（它会制造虚假的安全感）。

4. **监控你的监控系统**——Alertmanager 集群健康、Gossip 状态和通知速率应成为标准可观测性栈的一部分。如果监控系统本身出问题，你就是在盲飞。

5. **重载时 `--data.dir` 至关重要**——重载 Alertmanager 时，确保数据目录一致。使用不同的 `--data.dir` 重载等同于以空的静默存储重启。

6. **对控制平面进行 NetworkPolicy 审计**——Kubernetes 控制平面组件需要稳定的网络连接。一条无意中阻断 Gossip 端口的 `NetworkPolicy` 可以在任何有状态分布式系统中引发脑裂。

---

## 总结

### 时间线回顾

| 时间 | 事件 |
|------|------|
| 02:45 | 跨 50 个节点触发滚动更新 |
| 02:47 | PagerDuty 告警风暴开始——每分钟 50+ 条通知 |
| 02:50 | Slack `#alerts` 频道被刷屏 |
| 02:52 | 工程师查看 Alertmanager 界面——200+ 个独立告警组 |
| 02:55 | 工程师发现静默规则不匹配 |
| 03:00 | 200+ 条 PagerDuty 通知，500+ 条 Slack 消息 |
| 03:05 | SRE 主管介入——开始结构化排查 |
| 03:20 | 定位根因 |
| 03:25 | 部署应急修复：重新创建静默规则、修复分组、更新网络策略 |
| 03:30 | 通知量降至约 5/min |
| 03:45 | 宣布事件解决 |

### 配置对比

```yaml
# 修复前——存在问题的配置
route:
  group_by: ['...']
  group_wait: 5s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
```

```yaml
# 修复后——正确的配置
route:
  group_by: ['alertname', 'namespace', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
    - match:
        severity: warning
      receiver: slack-warnings
```

### 关键指标

| 指标 | 修复前 | 修复后 |
|------|--------|--------|
| 每分钟通知数 | 2000 | ~5 |
| 独立告警组数 | 500+ | 5-10 |
| 集群失败对等节点 | 2 | 0 |
| 静默匹配率 | ~0%（已损坏） | 100% |
| PagerDuty 限流触发 | 是 | 否 |
| 工程师寻呼疲劳度 | 严重 | 正常 |

### 最终结论

这次事故并非由单一故障引起，而是由**一系列相互叠加的配置错误**造成的：错误的分组策略、不足的批处理窗口、丢失的静默状态、静默规则与告警之间的标签漂移，以及导致集群分裂的网络策略。单独看每个问题都尚可管控，但叠加在一起就造成了最难熬的值班体验之一。

修复方案本身很简单——显式的 `group_by` 标签、更长的 `group_wait`、静默规则的基础设施即代码、以及集群健康监控——但需要一个痛苦的 incident 来推动它们的优先级。正如许多 SRE 教训一样，在凌晨三点被 PagerDuty 疯狂报警中学会的课程，代价最为昂贵。

---

*来自 SRE 团队的一手记录。名称和部分细节已做泛化处理，但技术根因与文档记录一致。*
