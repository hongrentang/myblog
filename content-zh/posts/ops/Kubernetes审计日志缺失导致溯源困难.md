---
title: "Kubernetes 审计日志缺失导致溯源困难——集群被攻破后安全团队无法追踪攻击路径"
date: 2026-06-03
weight: 100320
slug: "audit-log-missing-forensics"
tags: ["kubernetes", "security", "audit", "forensics", "troubleshooting"]
categories: ["Security"]
description: "一次因 Kubernetes 审计日志配置不当导致安全事件溯源彻底失败的案例分析——审计策略级别过低、日志未持久化、无集中采集，攻破后安全团队两眼一抹黑"
keywords: "Kubernetes 审计日志缺失, K8s 审计策略配置错误, 集群安全事件溯源, 审计日志级别设置, Kubernetes 安全事件响应"
draft: false
featured: true
cover:
  image: ""
  caption: "Kubernetes 审计日志缺失——安全事件溯源受阻"
---

# Kubernetes 审计日志缺失导致溯源困难——集群被攻破后安全团队无法追踪攻击路径

## Common Search Queries

- kubernetes 审计日志未生效
- k8s 审计策略配置错误
- kubernetes 安全事件溯源没有日志
- 审计日志级别 Metadata 和 RequestResponse 区别
- kubernetes 审计日志采集 fluentd
- kube-apiserver 审计日志最佳实践
- kubernetes 应急响应无日志可用

---

## 故障经过 / The Incident

**环境**: K8S v1.28, kubeadm 部署, 3 个控制平面节点 (HA), 50 个 Worker 节点, 本地数据中心。审计日志仅存储在控制平面本地磁盘, 未配置外部日志采集。

**时间**: 周六凌晨 03:14。PagerDuty 告警: `NodePortExhaustion` 同时在 12 个 Worker 节点上触发。紧接着 `CPUThrottlingHigh` 告警出现在 `kube-system` 命名空间。然后是 `KubeAPILatencyHigh`——API Server 响应时间从 50ms 飙升到 12 秒。

**初步发现**: 值班工程师登录控制平面, 发现 `kube-system` 命名空间中运行着一个名为 `kube-proxy-monitor` 的加密货币挖矿容器:

```bash
# 工程师看到的异常 Pod
kubectl get pods -n kube-system | grep -v Running
NAME                               READY   STATUS    RESTARTS   AGE
kube-proxy-monitor                 1/1     Running   0          2h
```

```bash
# Describe 显示异常镜像来源
kubectl describe pod kube-proxy-monitor -n kube-system | grep Image
Image:          docker.io/malregistry/kube-proxy-monitor:v1.0.0
```

该镜像并非来自组织内部镜像仓库。Pod 已运行 2 小时, 正在消耗 8 个 CPU 核心和 6 GB 内存进行挖矿。

**影响范围**:
- 集群整体性能下降 60%
- 12 个 Worker 节点端口耗尽——关键入站流量丢失
- API Server 响应延迟飙升, 导致 Controller Manager 选主失败
- 挖矿程序持续运行约 2 小时才被发现
- **攻击者的完整入侵路径无法追溯**

---

## 背景 / Background

该集群 14 个月前使用 kubeadm 部署。当时审计日志是"可有可无"的配置项。团队启用了一个最小策略, 理由是"集群在防火墙后面""我们有网络策略"。

### 原始审计策略

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata   # ← 仅记录元数据, 不记录请求/响应体
  resources:
  - group: ""        # 核心 API 组
    resources: ["pods", "services", "configmaps", "secrets"]
- level: None        # ← 忽略所有其他资源
  resources:
  - group: ""
    resources: ["events", "endpoints"]
- level: None
  users: ["system:kube-controller-manager", "system:kube-scheduler"]
```

**该策略的关键缺陷:**
1. `level: Metadata` —— 仅记录用户、时间戳和资源名, **不记录请求体或响应体**。在溯源场景下, 只能看到*创建了某个 Pod*, 但看不到 Pod 的完整定义
2. `level: None` 排除了 controller-manager 和 scheduler 的所有操作——如果某个组件被攻陷, 其异常 API 调用完全不可见
3. 没有覆盖 `ClusterRole`/`ClusterRoleBinding` 资源——RBAC 篡改操作没有任何审计记录
4. 缺少兜底规则——任何未显式列出的资源类型**都不会生成审计日志**

### 日志存储配置

```bash
# kube-apiserver 配置片段
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-maxage=7       # ← 日志 7 天后删除
    - --audit-log-maxbackup=5     # ← 最多保留 5 个轮转文件
    - --audit-log-maxsize=100     # ← 每个文件 100 MB 后轮转
```

日志**仅**存储在控制平面节点的本地磁盘上, 没有配置集中式日志采集。`--audit-log-maxage=7` 意味着超过 7 天的日志会被自动删除。

### 问题所在

安全团队在发现入侵后立即去查找审计日志, 结果如下:

```bash
# 确认审计日志是否启用
ls -la /var/log/kubernetes/audit.log
-rw------- 1 root root 0 Jun 3 03:20 /var/log/kubernetes/audit.log
# 文件是空的——在应急响应过程中, API Server 重启导致日志轮转, 新文件从 03:20 开始记录
```

```bash
# 检查轮转日志
ls -la /var/log/kubernetes/audit.log.*
audit.log.20260602   # ← 只有 1 个轮转文件, 来自昨天
audit.log.20260601   # ← 以及前天的 1 个文件
```

仅剩 2 天的轮转日志。攻击发生在 2 小时前, 但关键时间窗口的审计日志**完全丢失了**——因为应急响应的第一步重启了 API Server。

---

## 排查过程 / Investigation

### 第 1 步: 确认审计日志状态

```bash
# 在控制平面节点上
kubectl -n kube-system get pods -l component=kube-apiserver -o jsonpath='{.items[0].spec.containers[0].command}' | jq .
```

```json
[
  "kube-apiserver",
  "--audit-log-path=/var/log/kubernetes/audit.log",
  "--audit-policy-file=/etc/kubernetes/audit-policy.yaml",
  "--audit-log-maxage=7",
  "--audit-log-maxbackup=5",
  "--audit-log-maxsize=100"
]
```

审计日志确实是"启用"的——但有着严重的局限性。

### 第 2 步: 审查审计策略

```bash
cat /etc/kubernetes/audit-policy.yaml
```

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["pods", "services", "configmaps", "secrets"]
- level: None
  resources:
  - group: ""
    resources: ["events", "endpoints"]
- level: None
  users: ["system:kube-controller-manager", "system:kube-scheduler"]
```

安全团队立即发现以下问题:
- `level: Metadata` 仅记录 `User`、`Timestamp`、`Verb`、`Object`、`SourceIPs`——没有请求体内容
- RBAC 变更操作 (`ClusterRole`/`ClusterRoleBinding`) **不在规则列表中**——它们会穿透到默认行为, 而默认行为是……不记录

```bash
# 审计日志的默认级别是什么?
# 根据 Kubernetes 官方文档: 如果没有匹配的规则, 请求不会被记录
```

这是关键缺口。攻击者完全可以创建一个 ClusterRoleBinding, 而审计日志中**不会有任何记录**。

### 第 3 步: 检查现有日志内容

```bash
# 在现有日志中搜索攻击时间窗口
grep "03:00" /var/log/kubernetes/audit.log.* 2>/dev/null | head -5
# 无输出——03:00-03:20 的日志在 API Server 重启后丢失
```

```bash
# 查看我们还剩下什么
cat /var/log/kubernetes/audit.log.20260602 | wc -l
81234

cat /var/log/kubernetes/audit.log.20260601 | wc -l
120456
```

昨天和前天的日志还在, 但**攻击发生的那 2 小时窗口完全不存在了**。

### 第 4 步: 尝试从现有日志重建攻击路径

```bash
# 搜索 ClusterRoleBinding 变更
grep -i "clusterrolebinding" /var/log/kubernetes/audit.log.* | head -10
# 无匹配——ClusterRoleBinding 操作从未被审计
```

```bash
# 搜索可疑的 pod 创建操作
grep "create.*pods" /var/log/kubernetes/audit.log.20260602 | head -20
```

```
2026-06-02T22:15:00Z  admin@company.com  create  pods  kube-system  deployment=...
2026-06-02T22:30:00Z  admin@company.com  create  pods  default  deployment=...
2026-06-02T23:45:00Z  system:node:cp-1  create  pods  kube-system  kube-proxy-monitor
```

等等——这里有一条关键记录。`system:node:cp-1` 在 23:45 创建了 `kube-proxy-monitor` Pod, **比告警触发早了 3 小时**。`system:node:cp-1` 身份意味着 API 调用使用的是 cp-1 节点的 kubelet 凭证。

```bash
# 检查 cp-1 的 kubelet 日志
journalctl -u kubelet -n 200 --no-pager | grep -i "kube-proxy-monitor"
```

```
Jun 2 23:44:58 cp-1 kubelet[2345]: E2345 api.go:123] Failed to create pod kube-proxy-monitor: node "cp-1" not found
Jun 2 23:44:59 cp-1 kubelet[2345]: I2345 kubelet.go:201] SyncLoop (PLEG): event for pod kube-proxy-monitor
Jun 2 23:45:01 cp-1 kubelet[2345]: I2345 kubelet.go:201] SyncLoop (PLEG): event for pod kube-proxy-monitor
```

kubelet 日志确认了这个 Pod 正在被 kubelet 管理, 但无法解释**是谁告诉 kubelet 创建它的**——因为正常情况下 kubelet 不会直接创建 Pod。

```bash
# 检查 kubelet 配置中的静态 Pod 路径
grep -i "staticpodpath\|staticPodPath\|--pod-manifest-path" /var/lib/kubelet/config.yaml
```

```
staticPodPath: /etc/kubernetes/manifests
```

```bash
# 检查静态 Pod 目录中是否有异常文件
ls -la /etc/kubernetes/manifests/
total 28
drwxr-xr-x 2 root root 4096 Jun  2 23:44 .
drwxr-xr-x 4 root root 4096 Jun  2 10:00 ..
-rw------- 1 root root 2002 Jun  2 10:00 kube-apiserver.yaml
-rw------- 1 root root 1856 Jun  2 10:00 kube-controller-manager.yaml
-rw------- 1 root root 1440 Jun  2 10:00 kube-scheduler.yaml
-rw-r--r-- 1 root root  422 Jun  2 23:44 kube-proxy-monitor.yaml   # ← 攻击者留下的
```

攻击者直接在控制平面节点的 `/etc/kubernetes/manifests/` 目录中写入了一个 **静态 Pod 清单文件**。kubelet 会监控此目录, 自动创建其中的任何 Pod 清单。这种方法完全绕过了 API Server——**不会产生任何审计日志记录**, 因为 Pod 从未通过 API 创建。

### 第 5 步: 确定攻击者如何获得 Shell 访问权限

```bash
# 检查 SSH 认证日志
grep "Accepted\|Failed" /var/log/auth.log | tail -50
```

```
Jun  2 23:30:01 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:05 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:09 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:12 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:15 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:18 cp-1 sshd[12345]: Accepted password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:18 cp-1 sshd[12345]: lastlog: control process exited
```

SSH 暴力破解成功。攻击者通过 SSH 获取了控制平面节点的 root 访问权限, 然后部署了静态 Pod 清单。但安全团队仍然有以下疑问:

- `10.0.0.99` 是已知的内部 IP 还是外部 IP?
- 攻击者如何知道 root 密码?
- 这是单节点被入侵还是多个节点受到影响?
- 攻击者是否在集群内进行了横向移动?

由于没有完整的审计日志, 这些问题都无法回答。

### 第 6 步: 检查其他两个控制平面节点

```bash
for node in cp-2 cp-3; do
  echo "=== $node ==="
  ssh $node "ls -la /etc/kubernetes/manifests/; echo '---'; grep 'kube-proxy-monitor' /var/log/kubernetes/audit.log 2>/dev/null"
done
```

```
=== cp-2 ===
total 24
drwxr-xr-x 2 root root 4096 Jun  2 10:00 .
drwxr-xr-x 4 root root 4096 Jun  2 10:00 ..
-rw------- 1 root root 2134 Jun  2 10:00 kube-apiserver.yaml
-rw------- 1 root root 1856 Jun  2 10:00 kube-controller-manager.yaml
-rw------- 1 root root 1440 Jun  2 10:00 kube-scheduler.yaml
(cp-2 无异常)

=== cp-3 ===
total 24
(无异常文件, grep 无结果)
```

仅 cp-1 被攻陷。但由于没有集中的审计日志, 团队无法确定攻击者是否从 cp-1 发起了任何 API 调用。

### 第 7 步: 从仅存的审计数据中提取信息

```bash
# 提取攻击者通过 kubelet 进行的 API 活动
cat /var/log/kubernetes/audit.log.20260602 | jq 'select(.user.username | test("system:node")) | {user: .user.username, verb: .verb, objectRef: .objectRef, sourceIPs: .sourceIPs, requestReceivedTimestamp: .requestReceivedTimestamp}' 2>/dev/null | head -30
```

```json
{
  "user": "system:node:cp-1",
  "verb": "create",
  "objectRef": {
    "resource": "pods",
    "namespace": "kube-system",
    "name": "kube-proxy-monitor"
  },
  "sourceIPs": ["127.0.0.1"],
  "requestReceivedTimestamp": "2026-06-02T23:45:01Z"
}
```

来源 IP 是 `127.0.0.1`——即本机。API 调用来自 cp-1 上的本地 kubelet, 这与静态 Pod 清单的行为一致。但由于 `level: Metadata`, **请求体**(Pod 定义的具体内容)**未被记录**。`responseStatus` 也未记录, 因此团队仅凭审计记录无法判断 API 调用是否成功。

---

## 根因 / Root Cause

1. **审计日志未对所有资源开启**: 审计策略仅覆盖了部分资源, 且级别为 `Metadata`。RBAC 资源 (`ClusterRole`、`ClusterRoleBinding`)、`Node` 操作和 CRD 均未纳入审计范围, 导致攻击者对权限的篡改完全不可见

2. **审计日志级别设为 `Metadata` 而非 `RequestResponse`**: `Metadata` 级别仅记录谁、什么时间、做了什么——但不记录请求体内容。这意味着攻击者创建的 Pod 定义、命令和 payload 均不可见。对于安全溯源场景, `RequestResponse` 级别是必需的, 因为它记录了完整的请求和响应负载

3. **审计日志仅存储在控制平面本地磁盘**: 未配置集中式日志采集 (SIEM 或对象存储)。当 API Server 在应急响应中被重启时 (这是常见的应急操作), 活动的审计日志文件被轮转/截断, 关键的 2 小时窗口日志被**永久丢失**

4. **日志保留周期过短**: `--audit-log-maxage=7` 意味着 7 天前的日志被自动删除。结合 `--audit-log-maxbackup=5`, 集群最多保留 500 MB 的审计日志。对于一个每小时生成数千 API 调用的 50 节点生产集群, 仅保留了最近几天的日志

5. **缺少审计日志异常告警**: 即使审计日志是完整的, 也没有任何消费系统来检测异常行为——比如"kubelet 在 kube-system 中创建了 Pod"或"凌晨 3 点出现新的 ClusterRoleBinding"

6. **攻击者完全绕过了 API Server**: 攻击者获取了控制平面节点的 root SSH 权限后, 直接向 `/etc/kubernetes/manifests/` 写入静态 Pod 清单。静态 Pod 由 kubelet 直接管理,**无需经过 API Server**, 因此无论审计策略如何配置, **都不会产生任何审计日志记录**。这是 Kubernetes 审计日志体系的一个盲区

### 审计日志级别对比

| 级别 | 记录内容 | 溯源价值 |
|-------|----------|----------|
| `None` | 不记录 | 无 |
| `Metadata` | 用户、时间戳、操作、资源对象、来源 IP、响应状态 | 低——知道*发生了什么*, 但不知道*具体内容* |
| `Request` | Metadata + 请求体 | 中——可以看到请求负载 |
| `RequestResponse` | Metadata + 请求体 + 响应体 | 高——可以看到完整的 API 交互过程 |

---

## 解决方案 / Resolution

### 应急处置 (立即执行)

```bash
# 1. 删除恶意静态 Pod
kubectl delete pod kube-proxy-monitor -n kube-system --force --grace-period=0
rm -f /etc/kubernetes/manifests/kube-proxy-monitor.yaml

# 2. 在防火墙层面封禁攻击者 IP
iptables -A INPUT -s 10.0.0.99 -j DROP

# 3. 轮转所有凭证——ServiceAccount、kubelet 证书
kubectl delete secret -n kube-system --all
# 使用新证书重新部署所有节点

# 4. 审计 SSH 访问——确认被攻陷的用户/密钥
grep "10.0.0.99" /var/log/auth.log
```

### 修复审计策略

```yaml
# /etc/kubernetes/audit-policy.yaml —— 正确配置
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# RBAC 变更 —— 必须记录请求和响应
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]

# Pod 操作 —— 需要看到完整的 Pod 定义
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods"]

# Secret 和 ConfigMap 操作 —— 敏感资源
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

# Node 操作
- level: Request
  resources:
  - group: ""
    resources: ["nodes"]

# 其他核心资源 —— Metadata 级别
- level: Metadata
  resources:
  - group: ""
    resources: ["services", "endpoints", "namespaces", "deployments", "statefulsets", "daemonsets"]

# 节点身份的操作 —— 监控机器行为的异常
- level: Metadata
  users: ["system:node:*"]
  userGroups: ["system:nodes"]

# 兜底规则 —— 记录所有其他操作
- level: Request
  omitStages:
  - "RequestReceived"
```

### 配置审计日志采集

```yaml
# fluentd-config.yaml —— 将审计日志投递到 S3/Elasticsearch
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-audit-config
  namespace: kube-system
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/kubernetes/audit.log
      pos_file /var/log/fluentd-audit.log.pos
      tag kubernetes.audit
      format json
      read_from_head false
    </source>

    <filter kubernetes.audit>
      @type record_transformer
      <record>
        cluster_name prod-cluster
        hostname "#{Socket.gethostname}"
      </record>
    </filter>

    <match kubernetes.audit>
      @type s3
      s3_bucket prod-k8s-audit-logs
      s3_region us-west-2
      path audit/${hostname}/
      <buffer>
        @type file
        path /var/log/fluentd-buffer
        timekey 3600
        timekey_wait 600
        timekey_use_utc true
      </buffer>
      <format>
        @type json
      </format>
    </match>
```

在存储层配置保留策略:

```bash
# S3 生命周期策略 —— 保留审计日志 365 天
aws s3api put-bucket-lifecycle-configuration \
  --bucket prod-k8s-audit-logs \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "audit-log-retention",
      "Status": "Enabled",
      "Filter": {"Prefix": "audit/"},
      "Expiration": {"Days": 365}
    }]
  }'
```

### 配置告警规则

```bash
# Elasticsearch 查询: 检测由节点身份创建的 Pod (非人为操作)
POST /k8s-audit-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.username": "system:node:" }},
        { "match": { "verb": "create" }},
        { "match": { "objectRef.resource": "pods" }}
      ]
    }
  }
}
```

```yaml
# Prometheus 告警: 审计日志采集延迟
- alert: AuditLogShippingLag
  expr: time() - fluentd_buffer_tail < 300
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "审计日志采集延迟 {{ $value }} 秒"

- alert: AuditLogMissing
  expr: absent(rate(apiserver_audit_event_total[5m]))
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "未检测到审计事件——审计日志可能已禁用或故障"
```

### 监控静态 Pod 目录

```yaml
# Falco 规则: 检测静态 Pod 清单目录的文件变更
- rule: Static Pod Manifest Change
  desc: 检测 /etc/kubernetes/manifests/ 下新增或修改的文件
  condition: >
    open_write and
    fd.name startswith /etc/kubernetes/manifests/ and
    not proc.name in (kubelet, kube-apiserver, kube-controller-manager, kube-scheduler)
  output: >
    静态 Pod 清单变更 (文件=%fd.name 进程=%proc.name 用户=%user.name)
  priority: CRITICAL
```

---

## 经验教训 / Lessons Learned

- **审计日志不是可选项**: 没有审计日志的 Kubernetes 集群等于蒙眼飞行。一旦集群被入侵, 将无法确定攻击向量、入侵入口或影响范围, 应急响应和溯源分析形同虚设

- **`Metadata` 级别对安全来说远远不够**: Kubernetes 官方文档推荐生产环境使用 `Metadata` 级别, 但对于安全敏感的环境, `RequestResponse` 是必需的。没有请求体, 你就看不到攻击者实际做了什么——只能看到时间戳和资源名

- **静态 Pod 是审计盲区**: 任何具有控制平面 root 权限的攻击者都可以部署静态 Pod, 完全绕过 API Server。仅靠审计日志是不够的——还需要在 `/etc/kubernetes/manifests/` 上配置文件完整性监控 (FIM) 和严格的 SSH 访问控制

- **日志采集和日志生成同样重要**: 仅本地存储加 7 天轮转意味着日志在一周内消失。如果控制平面节点宕机, 所有日志都会丢失。必须将日志投递到集中存储 (S3、Elasticsearch 或 SIEM), 保留至少 1 年以满足合规和溯源需求

- **API Server 重启销毁了关键证据**: 应急响应时团队重启了 kubelet 和 API Server——这轮转了活动的审计日志文件, 销毁了关键证据。应急响应手册**必须包含"在重启服务前保全日志"的步骤**

- **控制平面的 SSH 访问是特权操作**: 攻击者通过 SSH 获得了 root 权限。控制平面节点的 SSH 访问应通过堡垒机进行限制, 使用硬件密钥认证, 并至少启用多因素认证

- **不仅要告警日志内容, 还要告警日志缺失**: 审计事件的缺失可能意味着配置错误或攻击者关闭了日志。当审计事件数量降为零时应触发告警

---

## 总结 / Summary

攻击链:

```
SSH 暴力破解控制平面节点 cp-1 (root 密码被猜中)
→ 攻击者在 23:30 获得 cp-1 的 root Shell 权限
→ 向 /etc/kubernetes/manifests/kube-proxy-monitor.yaml 写入静态 Pod 清单
→ Kubelet 检测到新清单并自动创建 Pod —— 完全绕过 API Server
→ Pod 运行加密货币挖矿程序, 持续消耗 8 CPU + 6 GB 内存, 运行 2 小时
→ 03:14 CPUThrottlingHigh + NodePortExhaustion 告警触发
→ 值班工程师为应急重启了 API Server —— 活动审计日志被销毁
→ 安全团队发现审计日志在攻击时间窗口内为空
→ 在磁盘上找到静态 Pod 清单 —— 确认攻击向量
→ 但由于审计日志缺失: 无 RBAC 审计记录、无 API 调用历史、无法判断横向移动
```

**关键数据:**
- 发现用时: 3.7 小时 (23:45 部署 → 03:14 告警)
- 应急处置用时: 30 分钟
- 可用溯源证据: 10% (仅有静态 Pod 清单文件和 auth.log 记录)
- 影响: 集群性能下降 60%, 12 个 Worker 节点端口耗尽, 数据泄露风险未知

**如果能重来:**
- 对关键资源启用 `RequestResponse` 审计级别
- 配置集中日志采集到 S3, 保留 365 天
- 对 `/etc/kubernetes/manifests/` 实施文件完整性监控
- 通过堡垒机 + MFA 管控控制平面 SSH 访问
- 应急响应手册: **在任何组件重启前先保全日志**

这个集群确实"启用"了审计日志——但那是"合规检查"级别的审计, 不是"安全"级别的审计。这之间的差距, 让团队失去了了解攻击者如何入侵、做了什么、是否还在内部的唯一机会。
