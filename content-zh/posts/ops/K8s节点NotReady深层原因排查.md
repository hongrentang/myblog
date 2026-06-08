---
title: "Kubernetes 节点 NotRead — 当节点宕机却无人告诉你原因"
date: 2026-06-08
weight: 100420
slug: "node-notready-investigation"
tags: ["kubernetes", "troubleshooting", "node", "kubelet", "network"]
categories: ["Troubleshooting"]
description: "Kubernetes 节点 NotReady 深度排查实战 —— 三台 Worker 节点同时 NotReady 的系统性根因分析，涵盖 kubelet 挂起、CNI 故障、磁盘压力和容器运行时死锁"
keywords: "kubernetes node notready, kubelet notready, node status unknown, kubectl get nodes notready, kubernetes 节点故障排查"
draft: false
featured: true
cover:
  image: ""
  caption: "Node NotReady — 故障排查"
---

## 常见搜索词 (Common Search Queries)

| 搜索词 | 搜索意图 |
|---|---|
| `kubectl get nodes notready` | 查看异常节点 |
| `kubernetes node status unknown` | 理解节点生命周期状态 |
| `kubelet not reporting status` | 调试 kubelet 心跳失败 |
| `node.kubernetes.io/unreachable:NoSchedule` | 污点驱逐 Pod |
| `kubernetes node disk pressure` | 节点资源压力条件 |
| `crictl ps not working` | 容器运行时死锁 |
| `calico node crash loop back off` | CNI 插件故障 |
| `kubelet hung d state` | Kubelet 陷入不可中断睡眠 |
| `kubernetes node-monitor-grace-period` | Controller Manager 节点监控 |
| `kubelet volume mount stuck` | PVC 挂载导致 kubelet 挂起 |

---

## 故障经过 (The Incident)

### 环境配置

| 组件 | 版本 / 详情 |
|---|---|
| Kubernetes | v1.28 |
| 节点数 | 30 个 Worker 节点, 3 个 Control-plane 节点 |
| Pod 数量 | ~500 个 |
| CNI | Calico v3.26 |
| 容器运行时 | containerd 1.6.x |
| 操作系统 | Ubuntu 22.04 LTS |
| 部署环境 | 私有云裸金属服务器 |
| 监控系统 | Prometheus + Grafana + PagerDuty |

### 时间线

周二凌晨 03:17，值班工程师的手机被 PagerDuty 告警唤醒：**3 个节点 NotReady**。几分钟内，告警级联爆发 —— 受影响节点上的 Pod 卡在 `Terminating` 状态，其余 27 个节点被迫吸收重新调度的负载，CPU 和内存分配率飙升至 85% 以上。

初始症状：

- `kubectl get nodes` 显示三个 Worker 节点状态为 `NotReady`
- 这些节点上的所有 Pod 停留在 `Terminating` 或 `Unknown` 状态
- 剩余节点出现内存压力上升
- 多个关键服务的 SLI 开始下降
- PagerDuty 每 30 秒触发一次告警

```bash
# 03:17 时的屏幕输出
$ kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
node-01         Ready      <none>   342d   v1.28.4
node-02         Ready      <none>   342d   v1.28.4
node-03         Ready      <none>   342d   v1.28.4
node-04         NotReady   <none>   342d   v1.28.4   # <-- 异常
node-05         NotReady   <none>   342d   v1.28.4   # <-- 异常
node-06         NotReady   <none>   342d   v1.28.4   # <-- 异常
node-07         Ready      <none>   342d   v1.28.4
...
```

---

## 背景 (Background)

### Kubernetes 节点生命周期

理解节点为何变成 `NotReady`，需要掌握控制面如何跟踪节点健康状态。该机制包含三个组件：

1. **kubelet 心跳** —— 每个 `--node-status-update-frequency`（默认 10 秒），kubelet 向 API Server 提交一次 `NodeStatus` 更新，包含 `Ready`、`DiskPressure`、`MemoryPressure`、`PIDPressure`、`NetworkUnavailable` 等条件。

2. **Controller Manager 监控** —— `kube-controller-manager` 中的 `node-controller` 检查每个节点最后心跳的时间戳。如果在 `--node-monitor-grace-period`（默认 40 秒）内未更新，控制器将节点标记为 `Unhealthy`。如果超出 `--node-startup-grace-period`（默认 1 分钟，适用于新节点）或 `--node-monitor-grace-period`（适用于已有节点），条件变为 `Unknown`。

3. **基于污点的驱逐** —— 一旦节点变为 `NotReady` 或 `Unreachable`，controller-manager 会添加污点 `node.kubernetes.io/unreachable:NoSchedule`（或 `node.kubernetes.io/not-ready:NoSchedule`），阻止新 Pod 调度到该节点。在 `--pod-eviction-timeout`（默认 5 分钟）之后，受影响节点上的 Pod 被驱逐并重新调度到其他节点。

### 节点条件

Kubernetes 跟踪五种节点条件：

| 条件 | 描述 | 常见原因 |
|---|---|---|
| `Ready` | 节点健康，可接受 Pod | Kubelet 宕机、CNI 故障、运行时死锁 |
| `DiskPressure` | 节点磁盘空间不足 | 日志轮转失败、镜像拉取风暴 |
| `MemoryPressure` | 节点内存不足 | 工作负载超卖、内存泄漏 |
| `PIDPressure` | 节点进程数过多 | Fork 炸弹、僵尸进程 |
| `NetworkUnavailable` | 节点网络配置错误 | CNI 插件崩溃、iptables 损坏 |

---

## 排查过程 (Investigation)

### Step 1: 检查节点状态

第一个命令总是最直接的：

```bash
kubectl get nodes
kubectl describe node node-04
```

`describe node` 输出的 `Conditions` 部分包含了关键线索：

```
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Mon, 08 Jun 2026 03:10:17 +0000  Mon, 08 Jun 2026 02:55:03 +0000   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Mon, 08 Jun 2026 03:10:17 +0000  Tue, 02 Jun 2026 10:22:14 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         True    Mon, 08 Jun 2026 03:10:17 +0000  Mon, 08 Jun 2026 03:01:44 +0000   KubeletHasDiskPressure       kubelet has disk pressure
  PIDPressure          False   Mon, 08 Jun 2026 03:10:17 +0000  Tue, 02 Jun 2026 10:22:14 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                Unknown Mon, 08 Jun 2026 03:10:17 +0000  Mon, 08 Jun 2026 03:16:47 +0000   NodeStatusUnknown            Kubelet stopped posting node status.
```

三个突出问题：
1. **Ready = Unknown**，消息为 `Kubelet stopped posting node status` —— controller-manager 未收到心跳
2. **DiskPressure = True**，转换时间非常接近故障时间
3. 最后心跳时间 03:10:17，转换为 Unknown 的时间 03:16:47 —— 时隔约 6.5 分钟，远超 40 秒的宽限期

### Step 2: SSH 到节点检查 Kubelet

```bash
# SSH 到受影响节点
ssh node-04

# 检查 kubelet 状态
systemctl status kubelet
```

输出：
```
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2026-06-08 02:55:03 UTC; 24min ago
     Docs: https://kubernetes.io/docs/
 Main PID: 1837 (kubelet)
   Tasks: 23 (limit: 49152)
   Memory: 312.4M
   CGroup: /system.slice/kubelet.service
           └─1837 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml
```

Kubelet **正在运行** —— 没有崩溃，没有停止。那么为什么没有发送心跳呢？

```bash
journalctl -u kubelet --since "10 min ago" | tail -50
```

关键日志条目：

```
Jun 08 03:15:01 node-04 kubelet[1837]: E0608 03:15:01.234567   1837 kubelet_node_status.go:92] "Failed to update node status" err="Post \"https://api-server:6443/...\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"
Jun 08 03:15:01 node-04 kubelet[1837]: E0608 03:15:01.234678   1837 kubelet_node_status.go:96] "Node status update failed, will retry"
Jun 08 03:14:58 node-04 kubelet[1837]: I0608 03:14:58.123456   1837 reconciler.go:224] "VolumeReconciler: Volume operation stuck" volume="pvc-xxxxx" plugin="kubernetes.io/aws-ebs" state="mount" duration="5m32s"
```

最后一行是第一个真正线索：**卷协调器在挂载操作上卡住了**。

### Step 3: 检查 Syslog 中的 Kubelet 条目

```bash
grep -i "notready\|node.status\|heartbeat\|volume" /var/log/syslog | tail -20
```

输出显示一个重复模式 —— kubelet 的 API 客户端反复出现 `context deadline exceeded` 错误，所有这些错误都与一个无法完成的 PVC 挂载操作同时发生。

```
Jun 08 03:10:01 node-04 kubelet[1837]: I0608 03:10:01.123456   1837 operation_executor.go:110] "MountVolume operation started" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
Jun 08 03:10:01 node-04 kubelet[1837]: I0608 03:10:01.123789   1837 operation_executor.go:115] "MountVolume.WaitForAttach" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
Jun 08 03:10:01 node-04 kubelet[1837]: I0608 03:10:01.124000   1837 operation_executor.go:120] "MountVolume.MountDevice" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# ... 该挂载操作再无进展 ...
Jun 08 03:14:58 node-04 kubelet[1837]: I0608 03:14:58.123456   1837 reconciler.go:224] "VolumeReconciler: Volume operation stuck" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### Step 4: 检查容器运行时

```bash
crictl pods | head -10
```

该命令 **卡住了** —— 超过 30 秒没有返回。这是一个危险信号。

```bash
crictl ps -a | head -10
```

同样卡住。containerd 的 socket 没有响应：

```bash
ls -la /run/containerd/containerd.sock
# Socket 存在但操作超时
```

检查 containerd：

```bash
systemctl status containerd
```

```
● containerd.service - containerd container runtime
   Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2026-06-08 02:55:03 UTC; 24min ago
     Docs: https://containerd.io/
 Main PID: 1521 (containerd)
   Tasks: 45
   Memory: 1.2G
   CGroup: /system.slice/containerd.service
```

Containerd 正在运行，但其 gRPC socket 没有响应。高内存使用率（1.2 GB）表明 containerd 负载很大，很可能是在处理由崩溃循环的 CNI 触发的众多容器操作。

### Step 5: 检查 CNI (Calico)

从健康节点（或管理机器）执行：

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```

```
NAME                READY   STATUS             RESTARTS   AGE   IP              NODE
calico-node-abc12   1/1     Running            0          10d   192.168.1.10    node-01
calico-node-def34   0/1     CrashLoopBackOff   47         24m   192.168.1.11    node-04   # <--
calico-node-ghi56   0/1     CrashLoopBackOff   51         24m   192.168.1.12    node-05   # <--
calico-node-jkl78   0/1     CrashLoopBackOff   43         24m   192.168.1.13    node-06   # <--
calico-node-mno90   1/1     Running            0          10d   192.168.1.14    node-02
```

三个 NotReady 节点的 Calico Pod 全部处于 `CrashLoopBackOff`。查看 Calico 日志：

```bash
kubectl logs -n kube-system calico-node-def34 --previous | tail -30
```

关键错误：

```
2026-06-08 03:05:23.456 [ERROR][1] felix/iptables.go:789: Failed to update iptables rules: error executing iptables-restore: exit status 4 (iptables-restore: line 47: COMMAND_FAILED)
2026-06-08 03:05:23.789 [ERROR][1] felix/iptables.go:790: iptables-save output: # Warning: iptables-legacy tables present, use iptables-legacy-save to see them
2026-06-08 03:05:23.790 [ERROR][1] felix/iptables.go:791: Setting iptables to legacy mode -- iptables-legacy...
```

这表明 **iptables 规则损坏**。直接在节点上检查：

```bash
iptables -L -t nat | head -20
```

输出混乱 —— iptables 规则处于不一致状态，存在来自并发修改的重叠条目。

```bash
# 检查哪个进程在并发修改 iptables
auditctl -w /sbin/iptables -p x -k iptables_changes
ausearch -k iptables_changes --start today | tail -20
```

审计日志显示 `calico-node`（Felix）和另一个系统工具（一个遗留防火墙管理代理）同时写入 iptables 规则，导致竞态条件。

### Step 6: 检查磁盘压力

```bash
df -h /var/lib/kubelet
df -h /
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        98G   96G   2G   98% /           # <-- 根分区几乎占满
/dev/sdb1       500G  120G  380G  24% /var/lib/kubelet
```

根分区使用率达到 98%。磁盘压力条件真实存在。

```bash
du -sh /var/log/journal/
# 12G     /var/log/journal/
```

Journal 日志消耗了根分区 12 GB 空间。崩溃循环的 Calico Pod 产生了大量日志输出：

```bash
journalctl --disk-usage
# Archived and active journals take up 12.0G in the file system.
```

容器日志轮转也已失败：

```bash
ls -la /var/log/pods/
# 部分日志文件日期停留在数天前 —— 轮转未执行
```

检查 kubelet 驱逐配置：

```bash
cat /var/lib/kubelet/config.yaml | grep -i "eviction"
```

```
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
```

`nodefs.available: 10%` 阈值被突破（实际仅剩 2%），触发了 `DiskPressure` 条件。

### Step 7: 检查 Kubelet 配置

```bash
kubelet --version
# Kubernetes v1.28.4

cat /var/lib/kubelet/config.yaml | grep -i "nodeStatusUpdateFrequency\|heartbeat"
```

```
nodeStatusUpdateFrequency: 10s
```

默认 10 秒更新频率意味着 kubelet 本应每 10 秒发送一次心跳。但卡住的卷挂载导致 kubelet 内部 API 客户端阻塞，无法发送任何状态更新。

```bash
# 检查 D 状态进程（不可中断睡眠）
ps aux | grep " D"
```

```
root      1837  0.3  0.2  987654 32100 ?        D    03:10   2:34 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml
root      6201  0.0  0.0      0     0 ?        D    03:10   0:00 [kworker/u4:2]
```

Kubelet 进程处于 **D 状态**（不可中断睡眠），表明它被阻塞在一个 I/O 操作上 —— 这种情况下，是卡住的 NFS 后端 PVC 挂载。

---

## 根因 (Root Cause)

本次事故由 **三个并发问题** 同时爆发导致：

### 主要原因：Kubelet 因卷挂载而挂起

一个使用 **NFS 后端 PVC** 的 Pod 被调度到了 node-04。底层的 NFS 服务器变得无响应（后续追踪到影响特定 VLAN 的网络交换机固件问题）。Kubelet 的卷协调器启动挂载操作后，在等待 NFS 服务器响应时陷入 **D 状态**（不可中断睡眠）。

由于 kubelet 使用单线程事件循环处理状态更新（在后续版本的改进卷处理之前），被阻塞的挂载操作阻止了心跳 goroutine 向 API Server 上报状态。

### 次要原因：Calico CNI 崩溃循环

在所有三个受影响的节点上，Calico 的 Felix 代理因 **iptables 规则损坏** 而崩溃循环。调查发现，一个遗留的防火墙管理代理（之前安全工具部署的残留）也在通过 cron 定时任务修改 iptables 规则。当 Calico 和这个遗留工具同时进行 iptables 操作时，iptables 内核锁发生争用，导致 `iptables-restore` 失败并报错 `COMMAND_FAILED`。

每次 Calico 崩溃都会触发重启，崩溃循环产生大量日志输出 —— 加剧了磁盘压力问题。

### 第三原因：日志轮转失败引发的磁盘压力

根分区（`/dev/sda1` 占用 98%）是最终的触发因素。崩溃循环的 Calico Pod 用错误消息填满了 journal。同时，容器日志轮转停滞，原因是：

- `kubelet-eviction` 日志也在快速增长
- `journald` 没有配置大小限制（默认行为会消耗文件系统最多 10% 的空间）
- 当 `nodefs.available` 降至 10% 以下时，`DiskPressure` 条件被设置

### 连锁反应

```
03:10:00 -- NFS 服务器无响应
03:10:01 -- Kubelet 开始卷挂载，进入 D 状态
03:10:01 -- Kubelet 心跳停止
03:10:17 -- API Server 收到最后一次 NodeStatus 更新
03:10:?? -- Calico iptables 更新与遗留工具冲突，Felix 崩溃
03:11:00 -- Calico 开始崩溃循环，journal 被错误填满
03:12:00 -- Containerd 响应变慢（处理崩溃循环操作导致高内存）
03:13:00 -- CRI 操作开始超时
03:14:00 -- 检测到 DiskPressure（根分区 95%+）
03:14:58 -- Kubelet 卷协调器报告操作卡住
03:16:47 -- Node controller 将节点标记为 Unknown（node-monitor-grace-period 到期）
03:16:48 -- 应用污点 node.kubernetes.io/unreachable:NoSchedule
03:17:00 -- PagerDuty 告警触发
03:17:?? -- 受影响节点上的 Pod 标记为 Terminating，重新调度到其余节点
03:20:00 -- 剩余节点达到 85%+ 资源利用率，级联压力开始
```

---

## 解决方案 (Resolution)

### 应急响应（立即执行）

Step 1: **重启 kubelet**（在所有三个受影响节点上）

```bash
for node in node-04 node-05 node-06; do
    ssh $node "systemctl restart kubelet"
done
```

这打破了 D 状态挂起。Kubelet 进程被 systemd 终止后重新启动。重启后，卷协调器重试挂载操作，而此时 NFS 问题已独立解决（网络团队修复了交换机端口），挂载成功。

```bash
# 验证 kubelet 是否重新开始发送心跳
kubectl get nodes
# 所有节点应在 1-2 分钟内显示 Ready
```

Step 2: **强制删除卡住的 Pod**

```bash
# 识别受影响节点上卡在 Terminating 状态的 Pod
kubectl get pods --all-namespaces --field-selector spec.nodeName=node-04 | grep Terminating

# 强制删除
kubectl delete pod --force --grace-period=0 -n <namespace> <pod-name>
```

Step 3: **清理 journal 日志**以缓解磁盘压力

```bash
for node in node-04 node-05 node-06; do
    ssh $node "journalctl --vacuum-time=1d"
done
```

这释放了根分区约 8 GB 的空间。

```bash
# 验证磁盘压力是否缓解
df -h /
```

Step 4: **修复 Calico**，重启 DaemonSet Pod

```bash
kubectl delete pod -n kube-system -l k8s-app=calico-node --force --grace-period=0
```

同时，禁用引起 iptables 冲突的遗留防火墙代理：

```bash
for node in node-04 node-05 node-06; do
    ssh $node "systemctl stop legacy-firewall-agent && systemctl disable legacy-firewall-agent"
done
```

Step 5: **解除节点封锁**

```bash
kubectl uncordon node-04
kubectl uncordon node-05
kubectl uncordon node-06

# 验证 Pod 正在调度到恢复后的节点上
kubectl get pods -o wide | grep node-04
```

### 长期修复方案

| # | 措施 | 配置 |
|---|---|---|
| 1 | **添加所有 5 种节点条件的监控告警** | Prometheus: `kube_node_status_condition{condition="Ready",status="true"} == 0` |
| 2 | **配置磁盘压力驱逐阈值** | 在 kubelet 配置中设置 `evictionHard`: `nodefs.available: 5%`, `imagefs.available: 10%` |
| 3 | **限制 journald 日志大小** | `/etc/systemd/journald.conf`: `SystemMaxUse=5G`, `MaxFileSec=7day` |
| 4 | **为关键工作负载设置 PodDisruptionBudget** | `minAvailable: 2` 或 `maxUnavailable: 1` |
| 5 | **添加 kubelet 存活性监控** | 使用外部健康检查监控 `http://<node>:10248/healthz` |
| 6 | **移除遗留的 iptables 管理工具** | 在所有节点上停用 `legacy-firewall-agent` |
| 7 | **实施混沌工程测试** | 定期进行混沌实验：节点故障、网络分区、磁盘压力、CNI 终止 |
| 8 | **设置 Pod 驱逐超时时间** | 审查 `--pod-eviction-timeout`（默认 5 分钟）—— 对关键工作负载可适当减少 |

### Kubelet 配置加固

```yaml
# /var/lib/kubelet/config.yaml 追加配置
evictionHard:
  nodefs.available: 5%
  imagefs.available: 10%
  nodefs.inodesFree: 5%
evictionSoft:
  nodefs.available: 10%
evictionSoftGracePeriod:
  nodefs.available: 2m0s
evictionMaxPodGracePeriod: 60s
```

### Journald 配置

```ini
# /etc/systemd/journald.conf
[Journal]
SystemMaxUse=5G
SystemKeepFree=2G
MaxFileSec=7day
RuntimeMaxUse=1G
```

---

## 经验教训 (Lessons Learned)

| 问题 | 排查方式 | 修复方式 | 预防措施 |
|---|---|---|---|
| Kubelet 卷挂载 D 状态挂起 | `ps aux \| grep " D"` 显示 kubelet 处于不可中断睡眠 | `systemctl restart kubelet` | 添加 kubelet healthz 端点监控；NFS 挂载使用 `hard,intr` 选项；实现卷挂载超时 |
| CNI 因 iptables 损坏崩溃循环 | `kubectl get pods -n kube-system` 显示 calico-node CrashLoopBackOff | 停用遗留防火墙代理，删除 Calico Pod | 审计所有节点上竞争的 iptables 管理工具；统一使用 `iptables-legacy` 或 `iptables-nft` |
| Journal 日志导致磁盘压力 | `df -h /` 显示 98% 使用率；`journalctl --disk-usage` 显示 12 GB | `journalctl --vacuum-time=1d` 释放 8 GB | 用 `SystemMaxUse=5G` 限制 journald；设置日志轮转告警；搭建日志聚合（Loki/Elasticsearch） |
| Containerd 无响应 | `crictl pods` 无限期挂起 | 重启 kubelet（通过 CRI 触发 containerd 重启） | 设置 containerd 内存限制；监控 containerd gRPC 延迟 |
| Controller Manager 标记节点 NotReady | 默认 `node-monitor-grace-period: 40s` | 已有配置 —— 按设计正常工作 | 审查并可能调整宽限期；但本例中设置工作正常 |
| PagerDuty 告警洪泛 | 5 分钟内 30+ 条告警 | 确认、分级、解决 | 在 PagerDuty 中设置告警去重和速率限制 |
| 级联资源压力 | Prometheus 显示存活节点 CPU/内存 85%+ | 在级联导致故障前事故已解决 | 实施 cluster-autoscaler；使用拓扑分布约束；为每个命名空间设置资源配额 |

---

## 总结 (Summary)

### 事件链（可视化）

```
NFS 服务器故障
       │
       ▼
Kubelet 卷挂载（D 状态）──► 心跳停止
       │
       ├──► Controller-manager: node-monitor-grace-period 到期（40 秒）
       │         │
       │         ▼
       │    节点 → NotReady / Unknown
       │         │
       │         ▼
       │    污点: node.kubernetes.io/unreachable:NoSchedule
       │         │
       │         ▼
       │    Pod 驱逐 → 重新调度到健康节点
       │         │
       │         ▼
       │    级联资源压力影响剩余节点
       │
       ├──► Calico Felix 崩溃（iptables 竞态条件）
       │         │
       │         ▼
       │    CNI 崩溃循环 → Journal 填满 → DiskPressure
       │
       └──► Containerd 高内存 → CRI 操作超时
```

### 核心启示

本次 Node NotReady 事故是 **三种独立故障模式的汇聚** —— 卷挂载阻塞、CNI 因 iptables 损坏崩溃循环、以及日志溢出导致磁盘压力。任何一个单一问题都可以被处理，但它们的组合压垮了标准的恢复机制：

1. **Kubelet 挂起** 阻止了自动心跳恢复
2. **CNI 崩溃循环** 消耗了系统资源并填满了日志
3. **磁盘压力** 阻止了新的日志写入和容器操作

### 可操作建议

- **在 API Server 层面监控 kubelet 心跳** —— 不要仅依赖节点上的 `systemctl status`
- **在每个节点上设置 journald 日志限制** —— 默认无限制的 journal 增长会填满任何分区
- **审计冲突的系统代理** —— 多个工具管理 iptables、systemd 服务或 cron 任务往往导致隐蔽的故障
- **用混沌工程测试节点恢复流程** —— 重启 kubelet 很容易测试，但阻止成功恢复的事件链才是你需要在生产环境之前发现的
- **使用 PodDisruptionBudget** —— 它们不能防止节点故障，但能防止所有 Pod 同时撤离时发生的总工作负载崩溃
- **配置有足够提前量的 evictionHard 阈值** —— 在磁盘耗尽前留出反应时间

---

*发布日期: 2026-06-08 | 标签: kubernetes, troubleshooting, node, kubelet, network | 博客: https://blog.777157.xyz*
