---
title: "Pod 被驱逐排查实录——从 'NodeHasDiskPressure' 到理解 kubelet 驱逐机制"
date: 2026-05-22
weight: 100140
slug: "pod-eviction-kubelet-pressure"
tags: ["kubernetes", "k8s", "pod-eviction", "node-pressure", "troubleshooting"]
categories: ["K8S"]
description: "Pod 反复被 Evicted，排查从看日志、看节点资源到最终发现磁盘压力和日志轮转问题的完整过程"
keywords: "pod evicted, node pressure, kubelet eviction, disk pressure, container log rotation"
draft: false
featured: true
cover:
  image: "/images/pod-eviction-banner.svg"
  caption: "Pod Eviction 节点驱逐排查"
---

# Pod 被驱逐排查实录——从 "NodeHasDiskPressure" 到理解 kubelet 驱逐机制

## 问题现象

下午 3 点，监控报了一条告警："Deployment payment-worker 可用副本数低于期望值"。

上 K8S 看一下：

```bash
kubectl get pods -n payment
```

```
NAME                              READY   STATUS            RESTARTS   AGE
payment-worker-7d9f8c6b8f-abc12   0/1     Evicted           0          12m
payment-worker-7d9f8c6b8f-def34   0/1     Evicted           0          8m
payment-worker-7d9f8c6b8f-ghi56   0/1     Evicted           0          3m
payment-worker-7d9f8c6b8f-jkl78   1/1     Running           0          2m
payment-worker-7d9f8c6b8f-mno90   0/1     Evicted           0          15m
```

一连串 Evicted。3 个节点上的 payment-worker Pod 陆陆续续被驱逐，只剩最后一个刚拉起来的还在 Running。

> 环境：K8S 1.28，3 master + 5 worker，每个 worker 128GB 内存、20 核。paymnet-worker 是 CPU 密集型服务，每个 Pod 请求 2c4g。

**影响**：支付对账处理延迟从 5 分钟飙到 40 分钟，大量对账任务积压。用户层面暂时无感知（这是个异步 worker），但运营已经开始问"今天的对账单怎么还没出"。

## 排查过程

### 错误尝试 1：看 Pod 日志

第一反应——Pod 挂了，看看日志有什么线索。

```bash
kubectl logs -n payment payment-worker-7d9f8c6b8f-abc12
```

```
Error from server (BadRequest): previous terminated container "payment-worker" not found
```

Evicted 的 Pod 已经被 kubelet 干掉了，日志文件可能已经被清理。试试加 `--previous`：

```bash
kubectl logs -n payment payment-worker-7d9f8c6b8f-abc12 --previous
```

一样的结果。

**踩坑点**：Evicted 不是 CrashLoopBackOff，不是一个进程 crash 了等你看日志。它是**节点主动把你的 Pod 赶走了**，像房东收房一样——你的东西（日志）已经被清出去了。CrashLoopBackOff 的 Pod 还能看日志排查，Evicted 的 Pod 什么都没有。

### 错误尝试 2：看节点资源使用

觉得可能是节点资源不够。看看聚合情况：

```bash
kubectl top nodes
```

```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
worker-1   12500m       62%    78256Mi         61%
worker-2   9840m        49%    65432Mi         51%
worker-3   14200m       71%    90123Mi         70%
worker-4   6200m        31%    45200Mi         35%
worker-5   5000m        25%    32000Mi         25%
```

看起来还好？最高的 worker-3 也就 71% CPU、70% 内存，远没到打满的程度。内存用了 90GB，总 128GB，还剩 38GB。

这就奇怪了——资源很充裕，为什么 kubelet 要驱逐 Pod？

**踩坑点**：`kubectl top node` 只看 CPU 和内存。但 kubelet 的驱逐不仅看内存压力（MemoryPressure），还看**磁盘压力（DiskPressure）**和**PID 压力（PIDPressure）**。你只看了内存和 CPU，完全忽略了磁盘。

### 错误尝试 3：重部署 Pod

既然不确定原因，"重启试试"——把 Deployment 删了重建：

```bash
kubectl rollout restart deployment payment-worker -n payment
```

Pod 全部重建，调度到了不同节点。看起来恢复了。

但 2 小时后，同样的告警又来了。新的 Pod 又被驱逐了。

**踩坑点**：驱逐是节点级别的行为，重建 Pod 只是在绕路——如果根本问题（节点压力）没解决，新 Pod 调度到同一个节点一样会被驱逐。就像换了个租客住进一间有白蚁的房子，结果一样。

### 真正的发现：看节点状态和事件

知道问题在节点层面后，逐个检查节点：

```bash
kubectl describe node worker-3
```

在 Conditions 段看到了关键信息：

```
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  MemoryPressure       False   Thu, 22 May 2026 14:30:15 +0800   Thu, 22 May 2026 10:15:00 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         True    Thu, 22 May 2026 14:30:15 +0800   Thu, 22 May 2026 09:45:00 +0800   KubeletHasDiskPressure       kubelet has disk pressure
  PIDPressure          False   Thu, 22 May 2026 14:30:15 +0800   Thu, 22 May 2026 09:45:00 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
```

**`DiskPressure: True`**——磁盘有压力！

而且从 `LastTransitionTime` 看，DiskPressure 从早上 09:45 就已经开始了，持续了几个小时。这期间 kubelet 一直在尝试回收磁盘空间，包括驱逐 Pod。

再看看节点的 Event：

```bash
kubectl describe node worker-3 | grep -A 10 Events
```

```
Events:
  Normal   NodeHasSufficientMemory  5m    kubelet  Node worker-3 status is now: NodeHasSufficientMemory
  Normal   NodeHasDiskPressure      5h    kubelet  Node worker-3 status is now: NodeHasDiskPressure
  Normal   NodeHasSufficientDisk    10m   kubelet  Node worker-3 status is now: NodeHasSufficientDisk
  Normal   NodeHasDiskPressure      8m    kubelet  Node worker-3 status is now: NodeHasDiskPressure
```

磁盘压力在反复波动——一会儿有、一会儿没。kubelet 在阈值上下反复横跳（这在 kubelet eviction 里叫"抖动"）。

SSH 到节点上看一下：

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        98G   88G   5.4G  95% /
/dev/sdb        500G  200G  300G  40% /data
```

根分区已经用了 95%！

```bash
# 看看什么在占空间
du -sh /var/log/pods/
```

```
45G     /var/log/pods/
```

```bash
du -sh /var/lib/containers/
```

```
30G     /var/lib/containers/
```

容器日志占了 45GB，容器镜像和层文件占了 30GB。这两个加起来就已经 75GB 了。

```bash
# 看看日志最多的 Pod
du -sh /var/log/pods/*payment*
```

```
12G     /var/log/pods/payment_worker-abc123
8G      /var/log/pods/payment_worker-def456
...
```

payment-worker 的日志最大，单个 Pod 日志能到 12GB——这些 Pod 跑了几天，日志就攒了这么多，没有任何轮转（log rotation）。

```bash
# 验证 kubelet 的驱逐阈值
kubectl describe node worker-3 | grep -i evict
```

```
Allocatable:
  ephemeral-storage:  78312664572
...
```

默认的 kubelet 驱逐阈值：

```bash
cat /var/lib/kubelet/config.yaml | grep -i eviction
```

```
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

根分区（nodefs）只剩 5.4G（约 5.5%），低于 10% 的硬阈值——触发 DiskPressure，kubelet 开始驱逐 Pod。

**为什么被驱逐的是 payment-worker**：kubelet 驱逐 Pod 时有优先级排序，按 QoS 级别从低到高依次驱逐：

1. BestEffort（无资源限制）→ 最先被驱逐
2. Burstable（部分资源限制）→ 其次
3. Guaranteed（所有资源限制）→ 最后被驱逐

查看 payment-worker 的 Pod 配置：

```bash
kubectl get pod payment-worker-7d9f8c6b8f-abc12 -n payment -o yaml | grep -A 6 resources
```

```yaml
resources:
  requests:
    cpu: "2"
    memory: 4Gi
  limits:
    cpu: "4"
    memory: 8Gi
```

requests 和 limits 不一致 → **Burstable** QoS。这不是最低优先级，但集群里很多基础组件（coredns、calico-node）都是 Guaranteed，所以 payment-worker 成了"第二个被驱逐"的批次。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | 节点根分区磁盘使用率达 95%，触发 kubelet DiskPressure 驱逐阈值 |
| 根本原因 | 容器日志没有配置轮转（log rotation），运行几天后积累到 45GB |
| 触发条件 | 日志无限制增长 → 根分区超 90% → kubelet 开始驱逐 nodefs 上优先级最低的 Pod |
| 为什么波动 | kubelet 驱逐 Pod 后释放少量日志文件 → 磁盘短暂低于阈值 → 新 Pod 调度进来产生新日志 → 再次超阈值（抖动循环） |

kubelet 驱逐的完整流程：

```
容器日志写满 /var/log/pods/ 
  → nodefs 使用率 > 90%（默认 evictionHard）
  → kubelet 设置 NodeDiskPressure=True
  → kubelet 开始驱逐 Pod（按 QoS 优先级：BestEffort → Burstable → Guaranteed）
  → Pod 被逐出后，其日志文件被清理 → nodefs 短暂释放
  → 新 Pod 调度进来 → 日志再次积累 → 循环
```

这就是为什么 Pod 在被反复驱逐重建——不是 Deployment 的问题，是节点层面在循环驱逐腾空间。

## 解决方案

### 快速止血：清理日志和镜像

手工释放空间，让节点恢复健康：

```bash
# 1. 清理已驱逐 Pod 的残留日志
journalctl --vacuum-size=200M

# 2. 清理未被引用的容器镜像
crictl rmi --prune

# 3. 找到并清理大日志目录（慎用，确认哪些可以删）
find /var/log/pods/ -name "*.log" -size +100M -exec truncate -s 0 {} \;

# 4. 驱逐节点上的 Pod，让它彻底恢复
kubectl drain worker-3 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker-3
```

```bash
# 验证释放了多少
df -h /
```

```
/dev/sda1        98G   78G   15G   80% /
```

从 95% 降到了 80%。节点 DiskPressure 状态清除：

```bash
kubectl describe node worker-3 | grep DiskPressure
```

```
DiskPressure         False   ...
```

### 根本修复：配置容器日志轮转

容器日志不轮转是生产环境的大坑。配置 Docker/containerd 日志轮转：

```bash
# 对于 containerd（新版 K8S 默认）
cat >> /etc/containerd/config.toml << 'EOF'
[plugins."io.containerd.grpc.v1.cri".containerd]
  max_container_log_line_size = 16384

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".logging]
  max_size = "50MB"
  max_files = 5
EOF

systemctl restart containerd
```

对于 Docker 运行时：

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

```bash
cat >> /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
EOF

systemctl restart docker
```

**为什么日志轮转能防复发**：
- 限制单个日志文件 50MB，最多保留 5 个文件
- 单个容器最多产生 250MB 日志，对比之前能到 12GB
- 轮转后旧日志被压缩/删除，不会无限增长
- 这是"不让温度继续升高"，驱逐是"温度太高了报警洒水"——你要做的是前者

### 长期加固：kubelet 驱逐阈值调优

默认阈值太宽松了（10% 才驱逐），在生产环境应该更保守：

```yaml
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "15%"
  nodefs.inodesFree: "10%"
  imagefs.available: "20%"

# 驱逐触发后，等资源恢复到什么程度才算解除压力
evictionSoft:
  nodefs.available: "20%"
evictionSoftGracePeriod:
  nodefs.available: "2m"
```

加一个监控告警：

```bash
# Prometheus 告警规则 - 驱逐预警
# - node:node_filesystem_usage > 0.8  → 提前告警（不是等到 90% 才通知）
# - kubelet_node_status_condition{condition="DiskPressure",status="true"}  → 立即告警

# 验证节点日志量监控
# 每条 worker 节点上跑一个 node_exporter + disk usage alert
```

### 验证恢复

```bash
# 1. 确认节点 DiskPressure 已解除
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name} {.status.conditions[?(@.type=="DiskPressure")].status}{"\n"}{end}'
```

```
worker-1 False
worker-2 False
worker-3 False
worker-4 False
worker-5 False
```

```bash
# 2. 确认 Pod 全部 Running
kubectl get pods -n payment | grep -c Evicted
# 0

kubectl get pods -n payment
# 全部 Running

# 3. 验证日志轮转生效
# 检查日志文件大小
ls -lh /var/log/pods/payment_worker-*/payment-worker/*.log
# 应该没有超过 50MB 的

# 4. 支付对账延迟恢复（应用层面）
# 监控面板显示延迟从 40min 降回 5min
```

✅ **恢复确认**：
- 所有节点 DiskPressure=False
- 所有 Pod Running，不再出现 Evicted
- 日志文件控制在 50MB 以内
- 对账延迟恢复正常

## 教训总结

1. **Evicted 不等于 CrashLoopBackOff。** 这是两个完全不同的机制。CrashLoopBackOff 是 Pod 内部进程挂了，看日志能定位；Evicted 是节点主动驱逐 Pod，要查节点状态。一开始我跑到 Evicted Pod 上 `kubectl logs`，方向就错了。

2. **`kubectl top node` 不够全面。** 它只看 CPU 和内存。kubelet 驱逐的三个维度：内存、磁盘、PID——你漏看了磁盘就找不到根因。排查节点问题时，`kubectl describe node` 里的 **Conditions** 段是你第一眼要看的东西。

3. **容器日志轮转不是可选配置。** 很多集群初始化脚本漏了这一步，默认容器日志是**无限制增长**的。一个每天输出几百 MB 日志的服务，一周就能撑爆根分区。Docker 时代可以用 daemon.json 配，containerd 时代在 config.toml 里配，不管用什么运行时，**上线前必须配好日志轮转**。

4. **理解了 QoS 才能理解驱逐顺序。** 一个节点上几十个 Pod，被驱逐的不一定是资源吃最多的那个，而是 QoS 最低的那个。如果你的关键业务 Pod 是 Burstable 甚至 BestEffort，排查节点压力时要优先检查它们的状态——它们会是第一批牺牲品。

5. **"重启"不是答案，根因才是。** 重建 Deployment 让 Pod 重新调度，看起来好了，但两小时后又挂了。如果只是临时止血，你永远发现不了日志轮转这个根本问题。驱逐类的故障尤其如此——节点压力不解除，重建多少次都没用。
