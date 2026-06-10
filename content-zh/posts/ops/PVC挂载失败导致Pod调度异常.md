---
title: "PVC 挂载失败导致 Pod 调度异常——PersistentVolumeClaim 拒绝挂载，Pod 卡死在 Pending 状态"
date: 2026-06-10
weight: 100480
slug: "pvc-mount-failure-pod-not-scheduling"
tags: ["kubernetes", "storage", "pvc", "troubleshooting", "volume"]
categories: ["Troubleshooting"]
description: "一次 PVC 挂载失败事件——CSI 驱动版本不兼容与 fsGroup 策略缺失导致有状态服务 Pod 全部卡在 Pending 状态"
keywords: "kubernetes pvc 挂载失败, pod 卡住 pending, csi 驱动故障, pvc attach 超时, kubernetes 存储故障排查"
draft: false
featured: true
cover:
  image: ""
  caption: "PVC 挂载失败——故障排查指南"
---

## 常见搜索关键词

PVC 挂载失败 Pod 卡住 Pending, volume attachment timeout kubernetes, CSI 驱动注册失败, fsGroupPolicy CSIDriver, Pod Pending PVC 无法挂载, Kubernetes 存储故障排查, EBS CSI 驱动升级, volume already used by pod that is not running, node-driver-registrar 失败, Kubernetes volume attach detach 生命周期

---

## 故障经过

### 环境信息

| 组件 | 版本/规格 |
|-----------|----------|
| Kubernetes | v1.29 |
| AWS EBS CSI 驱动 | v1.20 |
| 有状态工作负载 | 20 个 StatefulSet |
| 存储类 | AWS EBS gp3 (WaitForFirstConsumer, Delete 回收策略) |
| 节点操作系统 | Amazon Linux 2 |
| 容器运行时 | containerd 1.7.x |
| 区域 | AWS ap-southeast-1 |

### 时间线

在一次计划维护窗口内，团队对 Kubernetes 控制平面进行了常规升级，从 v1.28 升级到 v1.29。升级本身顺利完成——所有控制平面组件（kube-apiserver、kube-controller-manager、kube-scheduler）均正常运行。节点镜像通过托管节点组刷新并行更新。

然而，节点组刷新完成后的几分钟内，值班工程师开始收到告警：

- 多个 StatefulSet Pod 卡在 Pending 状态
- 现有 StatefulSet 的滚动更新无法推进
- 引用 PVC 的新 Deployment 无法调度

### 症状表现

1. **Pod 状态——卡在 Pending**：所有引用了 PersistentVolumeClaim 的新创建 Pod 都无限期停留在 Pending 状态。执行 `kubectl get pods` 显示：

```
NAME                       READY   STATUS    RESTARTS   AGE
web-app-0                  0/1     Pending   0          12m
web-app-1                  0/1     Pending   0          12m
db-statefulset-0           0/1     Pending   0          15m
cache-0                    0/1     Pending   0          8m
```

2. **kubectl describe pod** 显示卷错误：

```
Events:
  Type     Reason              Age   From                     Message
  ----     ------              ----  ----                     -------
  Warning  FailedAttachVolume  45s   attachdetach-controller  Failed to attach volume "vol-xxxxxxxxxxxxx": rpc error: code = DeadlineExceeded desc = context deadline exceeded
  Warning  FailedMount         30s   kubelet                  Unable to attach or mount volumes: unmounted volumes=[data], unattached volumes=[data default-token-xxxx]: timed out waiting for the condition
```

3. **StatefulSet 滚动更新失败**：手动对 StatefulSet 执行滚动更新导致其卡死——新 Pod 卡住，旧 Pod 一直处于 Terminating 状态：

```
NAME                       READY   STATUS        RESTARTS   AGE
web-app-0                  0/1     Pending       0          2m
web-app-1                  1/1     Running       0          48m
web-app-2                  0/1     Terminating   0          48m
```

4. **PVC 状态——Pending**：PVC 本身显示 Pending 状态，没有任何卷附件信息：

```
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-web-app-0 Pending                                      gp3            12m
```

5. **Volume Already Used 错误**：部分 Pod 显示出另一种同样令人困惑的错误：

```
Warning  FailedAttachVolume  20s   attachdetach-controller  Volume is already used by pod "web-app-0" that is not running
```

---

## 背景

### Kubernetes 存储架构

Kubernetes 存储通过三个核心对象来管理：

- **PersistentVolume (PV)**：在集群中预置的存储资源，拥有独立于任何 Pod 的生命周期。PV 是集群资源。
- **PersistentVolumeClaim (PVC)**：用户对存储的请求。PVC 消费 PV 资源。声明可以请求特定的大小、访问模式和存储类。
- **StorageClass**：描述存储的"类别"。不同类别映射到不同的服务质量级别、备份策略或自定义制备程序——由 provisioner 字段决定。

流程是：用户创建 PVC → Kubernetes 找到匹配的 PV（或通过 StorageClass 动态制备一个）→ PVC 绑定到 PV → Pod 引用 PVC → 卷被挂载到 Pod。

### CSI 驱动模型

容器存储接口（CSI）是将存储系统暴露给容器工作负载的标准。CSI 驱动由两个主要组件构成：

1. **Controller Plugin（StatefulSet）**：以 Deployment 形式运行在集群中。负责卷的制备、挂载/卸载和快照操作。与云供应商 API 通信（例如，EBS 卷挂载需要与 AWS EC2 API 通信）。

2. **Node Plugin（DaemonSet）**：在每个节点上运行。负责卷的挂载、格式化和绑定到 Pod。包含三个容器：
   - `csi-driver`：实际的存储驱动，与存储后端交互
   - `node-driver-registrar`：边车容器，通过 kubelet 插件注册机制将 CSI 驱动注册到 kubelet
   - `liveness-probe`：CSI 驱动的健康检查

CSI 卷的完整生命周期：
1. **CreateVolume**：CSI 控制器预置存储卷（例如创建 EBS 卷）
2. **ControllerPublishVolume**：将卷挂载到节点（例如 EC2 AttachVolume）
3. **NodeStageVolume**：在节点上格式化并挂载卷
4. **NodePublishVolume**：将卷绑定挂载到 Pod 的文件系统
5. 卸载和删除时执行相反的操作

### 卷挂载/卸载生命周期

kube-controller-manager 中的 attach/detach 控制器负责管理卷挂载操作。当 Pod 被调度到节点时：

1. 调度器通知控制器需要将卷挂载到特定节点
2. 控制器调用 CSI 驱动的 ControllerPublishVolume
3. CSI 控制器插件调用云供应商 API 来物理挂载卷
4. 挂载完成后，节点上的 kubelet 发现设备并进行挂载

如果这个链中的任何一步失败，Pod 将卡在 Pending 状态。

### fsGroup 与 Pod 安全上下文

在 Kubernetes 中，`securityContext.fsGroup` 指定一个适用于 Pod 中所有容器的补充组。当卷被挂载时，kubelet 会递归更改卷内容的拥有权和权限以匹配该组。这对于需要对挂载卷有写入权限的有状态应用来说是必要的。

在 Kubernetes 1.28 及更高版本中，`fsGroup` 的行为发生了重大变化。以前，kubelet 会始终执行递归权限更改。从 1.28 开始，CSI 驱动必须在 CSIDriver 对象中显式声明其 `fsGroupPolicy`：

- `None`（默认值）：CSI 驱动不支持任何 fsGroup 策略。kubelet 在挂载时不会执行权限更改。
- `File`：CSI 驱动通过文件级权限更改支持 fsGroup。kubelet 会对卷执行递归的 chown/chmod。
- `ReadWriteOnceWithFSType`：CSI 驱动仅对非共享卷（访问模式为 ReadWriteOnce）和特定的文件系统类型支持 fsGroup。

如果未设置 `fsGroupPolicy` 或设置为 `None`，依赖 `fsGroup` 进行文件系统权限管理的有状态 Pod 将失败，因为 kubelet 会跳过权限修复。

---

## 排查过程

### 第 1 步：检查 Pod 状态

问题的第一个迹象是大量的 Pending Pod。执行：

```bash
kubectl get pods -A | grep Pending
```

发现跨多个命名空间有 40 多个 Pod 卡在 Pending 状态。所有这些 Pod 都使用了 PVC 挂载。

检查其中一个卡住的 Pod：

```bash
kubectl describe pod web-app-0
```

Events 部分显示：

```
Events:
  Type     Reason              Age   From                     Message
  ----     ------              ----  ----                     -------
  Warning  FailedAttachVolume  2m    attachdetach-controller  Failed to attach volume "vol-0a1b2c3d4e5f": rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

**关键观察**：挂载操作超时。控制器能够发起挂载，但 CSI 驱动未能在截止时间内响应。

### 第 2 步：检查 PVC/PV 状态

```bash
kubectl get pvc -A
```

输出显示所有新创建的 PVC 都处于 Pending 状态：

```
NAMESPACE   NAME              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
default     data-web-app-0    Pending                                      gp3            15m
default     data-web-app-1    Pending                                      gp3            12m
database    data-db-0         Pending                                      gp3            10m
```

查看特定 PVC：

```bash
kubectl describe pvc data-web-app-0
```

describe 输出显示 PVC 正在等待卷被创建——StorageClass 的制备程序还没有创建 EBS 卷。

### 第 3 步：检查 CSI 驱动 Pod

检查 CSI 驱动组件：

```bash
kubectl get pods -n kube-system -l app=ebs-csi-node
```

节点插件 DaemonSet 的 Pod 全部显示 Running：

```
NAME                    READY   STATUS    RESTARTS   AGE
ebs-csi-node-abc12      3/3     Running   0          6h
ebs-csi-node-def34      3/3     Running   0          6h
ebs-csi-node-ghi56      2/3     Running   0          6h
ebs-csi-node-jkl78      3/3     Running   0          6h
```

等等——有一个节点显示的是 `2/3` 而不是 `3/3`。这很可疑。CSI 节点插件的三个容器中有一个没有就绪。

```bash
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

控制器插件也在运行：

```
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-7d8f9c6b8c-ab12   2/2     Running   0          6h
ebs-csi-controller-7d8f9c6b8c-cd34   2/2     Running   0          6h
```

### 第 4 步：检查 CSI 驱动日志

检查显示 `2/3` 就绪状态的节点插件容器的日志：

```bash
kubectl logs -n kube-system ebs-csi-node-ghi56 csi-driver
```

日志显示重复的注册错误：

```
I0610 10:15:23.456789       1 main.go:114] Version: v1.20.0
I0610 10:15:23.456912       1 main.go:127] Driver: ebs.csi.aws.com
E0610 10:15:23.457123       1 server.go:82] Failed to get driver capabilities: node service capability not supported
E0610 10:15:23.457156       1 main.go:135] Failed to start driver: node service capability not supported
```

这是一个关键的发现——CSI 驱动版本太旧，不支持 K8s 1.29 所需的节点服务能力。

同时检查控制器插件日志：

```bash
kubectl logs -n kube-system ebs-csi-controller-7d8f9c6b8c-ab12 csi-provisioner
```

制备程序日志显示：

```
W0610 10:15:30.123456       1 connection.go:182] Still connecting to unix:///csi/csi.sock, error: connection error
I0610 10:15:30.789012       1 controller.go:144] CreateVolume: called with args: {name: pvc-xxxxxxxxxx}
E0610 10:15:30.789156       1 controller.go:155] Failed to create volume: rpc error: code = Unavailable desc = all SubConns are in TransientFailure
```

CSI 控制器无法与存储后端通信，这就解释了为什么卷没有被制备。

### 第 5 步：检查 CSIDriver 对象

```bash
kubectl get csidriver ebs.csi.aws.com -o yaml
```

这是排查过程中的关键时刻：

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com
spec:
  attachRequired: true
  podInfoOnMount: true
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
```

注意**缺失了什么**：没有 `fsGroupPolicy` 字段。在 K8s 1.29 中，如果未指定 `fsGroupPolicy`，则默认为 `None`。这意味着 kubelet 永远不会对挂载的卷执行权限修复，导致使用 `securityContext.fsGroup` 的 Pod 失败。

### 第 6 步：检查节点 CSI 注册

```bash
ls -la /var/lib/kubelet/plugins/ebs.csi.aws.com/
```

在某些节点上，CSI 插件套接字文件丢失：

```
total 0
```

在套接字存在的节点上，检查注册状态：

```bash
ls -la /var/lib/kubelet/plugins_registry/
```

node-driver-registrar 没有正确地将驱动注册到 kubelet，日志确认显示"node service capability not supported"。

### 第 7 步：检查 Kubelet 日志

```bash
journalctl -u kubelet | grep -i "volume\|pvc\|csi"
```

kubelet 日志确认了问题：

```
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: E0610 10:20:15.123456    1234 reconciler.go:256] volume_attachment "vol-0a1b2c3d4e5f" has error: "timeout waiting for attachment to complete"
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: E0610 10:20:15.789012    1234 kubelet_volumes.go:154] Could not mount volume "pvc-xxxxxxxxxx" (vol-0a1b2c3d4e5f): context deadline exceeded
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: W0610 10:20:15.789123    1234 volume_manager.go:487] Volume "pvc-xxxxxxxxxx" is not attached to node "ip-10-0-1-23"
Jun 10 10:20:15 ip-10-0-1-23 kubelet[1234]: E0610 10:20:16.123789    1234 plugin_manager.go:136] Registration of plugin "ebs.csi.aws.com" failed: node-driver-registrar reported error: driver responded with an error
```

注册失败和卷挂载超时明显是相关的。

### 第 8 步：检查 StorageClass

```bash
kubectl get sc gp3 -o yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

StorageClass 看起来是正确的——它指向正确的 CSI 制备程序，使用 WaitForFirstConsumer（意味着卷只在 Pod 被调度后才被制备），并且具有正确的文件系统类型。这确认 StorageClass 本身不是问题所在。

---

## 根因

排查揭示了三个相互关联的问题，都源于同一个根因：**CSI 驱动版本与 Kubernetes 1.29 升级不兼容**。

### 问题 1：CSI 驱动版本不匹配

集群运行的是 EBS CSI 驱动 `v1.20.0`，该版本在 K8s 1.29 可用之前发布。此版本没有实现 K8s 1.29 的 kubelet 所需的 CSI 节点服务能力。具体来说：

- `node-driver-registrar` 边车容器（包含在 CSI 驱动中）太旧，无法通过更新后的 kubelet 插件注册协议进行注册
- CSI 驱动的 `NodeGetInfo` RPC 没有返回 K8s 1.29 期望的能力
- 由于驱动未正确注册，`NodeStageVolume` 和 `NodePublishVolume` 调用失败

### 问题 2：CSIDriver 规格中缺少 fsGroupPolicy

Kubernetes 1.28 为 CSIDriver 对象引入了 `fsGroupPolicy` 字段。当创建或更新 CSIDriver 而没有此字段时：

- 在 K8s 1.27 及更早版本中：kubelet 默认处理 fsGroup 更改
- 在 K8s 1.28+ 中：该字段默认为 `None`，意味着 kubelet **跳过**所有 fsGroup 权限修复

EBS CSI 驱动 v1.20.0 没有设置 `fsGroupPolicy`，因为该字段在驱动开发时还不存在。K8s 升级后，CSIDriver 对象继续工作但没有此字段，但 kubelet 现在将缺失的字段解释为 `fsGroupPolicy: None`，这导致：

- 依赖 `securityContext.fsGroup` 获取文件系统写入权限的有状态 Pod 无法启动
- 卷挂载权限检查静默失败
- Pod 停留在 Pending 状态，因为挂载无法完成

### 问题 3：VolumeAttach 超时连锁反应

由于节点插件没有正确注册，attach/detach 控制器可以完成 `ControllerPublishVolume` 调用（该调用发出 EC2 API 请求将卷挂载到实例），但节点端的操作（发现设备、准备和发布挂载）会失败。这造成了一个连锁反应：

1. PVC 请求一个卷
2. CSI 控制器成功创建了 EBS 卷
3. attach/detach 控制器调用 ControllerPublishVolume，EBS 卷被挂载到 EC2 实例
4. 但由于 CSI 驱动未注册，节点上的 kubelet 无法发现或挂载设备
5. 挂载超时到期，Kubernetes 重试，但注册失败持续存在
6. Pod 卡在 Pending 状态

### "Volume Already Used" 错误

部分 Pod 显示 `"Volume is already used by pod that is not running"`。这是因为：

- 在 StatefulSet 滚动更新期间，旧的 Pod 被终止，但其 PVC 仍然处于绑定状态
- 新 Pod 被创建并试图挂载同一个 PVC
- attach/detach 控制器发现该卷仍然"挂载"在旧 Pod 上（旧 Pod 已被删除，但由于 CSI 驱动故障，卸载操作从未完成）
- 控制器拒绝将卷挂载到新 Pod 上，从而产生这个容易误导的错误消息

---

## 解决方案

### 紧急修复——升级 CSI 驱动

立即修复是将 EBS CSI 驱动升级到与 Kubernetes 1.29 兼容的版本。

```bash
# 部署兼容版本的 stable overlay
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=v1.30.0"
```

此命令：

1. 使用更新的镜像部署 CSI 驱动控制器（StatefulSet）
2. 使用更新的镜像在所有节点上部署 CSI 驱动节点插件（DaemonSet）
3. 使用新的规格更新 CSIDriver 对象（包括 `fsGroupPolicy: File`）
4. 应用必要的 RBAC 和 ServiceAccount 更新

升级后，验证 CSIDriver 对象：

```bash
kubectl get csidriver ebs.csi.aws.com -o yaml
```

更新后的输出现在显示：

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: ebs.csi.aws.com
spec:
  attachRequired: true
  fsGroupPolicy: File
  podInfoOnMount: true
  volumeLifecycleModes:
  - Persistent
  - Ephemeral
```

关键变化是 `fsGroupPolicy: File`——这告诉 kubelet 执行为有状态 Pod 所需的递归权限更改。

### 清理卡住的 Pod

升级 CSI 驱动后，需要清理积压的卡住 Pod：

```bash
# 删除所有 Pending 的 Pod——它们将由控制器重新创建
kubectl delete pods --field-selector=status.phase=Pending -A
```

这将触发 StatefulSet 和 Deployment 控制器重新创建 Pod。使用更新后的 CSI 驱动，新 Pod 能够成功挂载和装载卷。

### 即时补丁（备选方案）

如果无法立即执行完整的 CSI 驱动升级（例如需要审批），可以通过手动补丁 CSIDriver 对象快速解决：

```bash
kubectl patch csidriver ebs.csi.aws.com -p '{"spec":{"fsGroupPolicy":"File"}}'
```

这将立即修复 fsGroup 问题，但**不会**修复 node-driver-registrar 兼容性问题。节点插件仍然需要升级才能正确注册到 kubelet。

### 验证恢复

验证修复是否生效：

```bash
# 检查所有 Pod 是否运行
kubectl get pods -A | grep -v Running | grep -v Completed

# 检查 PVC 是否已绑定
kubectl get pvc -A | grep -v Bound

# 检查 CSI 节点 Pod 是否全部 3/3
kubectl get pods -n kube-system -l app=ebs-csi-node

# 执行测试滚动更新
kubectl rollout restart statefulset web-app
```

所有 Pod 转换为 Running 状态，PVC 显示 Bound 状态，滚动更新顺利完成。

### 长期预防

为防止此类问题再次发生，我们实施了以下措施：

**1. 升级检查清单补充**

在集群升级手册中增加了以下检查项：

```
升级前检查：
  - 验证 CSI 驱动版本与目标 K8s 版本的兼容性
  - 执行: kubectl get csidriver -o yaml
  - 验证 fsGroupPolicy 是否正确设置（File 或 ReadWriteOnceWithFSType）
  - 验证 CSI DaemonSet 中的 node-driver-registrar 镜像版本
  - 检查 Kubernetes 变更日志中与存储相关的重大更改（例如 fsGroupPolicy、树内存储插件移除）

升级后验证：
  - 创建测试 PVC 和 Pod，验证挂载成功
  - 验证 CSI 驱动注册: ls /var/lib/kubelet/plugins/<驱动名称>/
  - 执行完整的 StatefulSet 滚动更新端到端测试
  - 监控 volume_attach_status 和 csi_volume_op_total 指标
```

**2. 监控和告警**

添加 Prometheus 记录规则和告警：

```
# 告警：CSI 驱动注册失败
- alert: CSIDriverRegistrationFailure
  expr: rate(csi_plugin_registrations_total{status="failure"}[5m]) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "检测到 CSI 驱动注册失败"

# 告警：卷挂载超时
- alert: VolumeAttachTimeout
  expr: rate(volume_operation_total_seconds_sum{status="fail"}[5m]) > 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "卷挂载操作正在失败"

# 记录：CSI 驱动版本
- record: csi_driver_version
  expr: csv{driver="ebs.csi.aws.com"}
```

**3. CI/CD 流水线检查**

在部署流水线中添加版本兼容性验证：

```bash
# 在 K8s 升级前检查 CSI 驱动兼容性
verify_csi_compatibility() {
  local current_k8s_version="$1"
  local target_k8s_version="$2"
  local csi_driver_image="$3"

  # 从镜像元数据中检索 CSI 驱动支持的 K8s 版本
  # 如果 CSI 驱动未声明支持目标版本，则流水线失败
  echo "验证 CSI 驱动 $csi_driver_image 与 K8s $target_k8s_version 的兼容性..."
}
```

**4. 卷生命周期测试**

添加了升级后测试套件，验证完整的卷生命周期：

- 制备测试卷
- 将卷挂载到节点
- 将卷装载到 Pod
- 写入数据并验证 fsGroup 权限
- 卸载和分离卷
- 删除卷

此测试在每次 K8s 升级后自动在非生产环境中运行。

---

## 经验教训

### 哪里出了问题

1. **缺乏升级前兼容性验证**：升级前没有检查 CSI 驱动版本与目标 Kubernetes 版本的兼容性。简单的 `kubectl get csidriver` 命令和查看 CSI 驱动发布说明就可以发现不兼容问题。

2. **缺乏 fsGroupPolicy 意识**：团队不知道 K8s 1.28 引入了一个破坏性变更，要求 CSI 驱动显式声明 `fsGroupPolicy`。这在 Kubernetes 变更日志中有记录，但未纳入升级检查清单。

3. **升级后验证不充分**：升级后的检查仅验证了控制平面组件健康（kube-apiserver、controller-manager、scheduler）。没有包含有状态工作负载的卷生命周期测试。

4. **认为 CSI 升级跟随 K8s 升级**：CSI 驱动被视为"稳定的基础设施组件"，不需要随 Kubernetes 控制平面一起更新。实际上，CSI 驱动版本与 Kubernetes 版本紧密耦合。

### 做得好的地方

1. **系统化排查**：排查按照逻辑顺序推进：Pod 状态 → PVC 状态 → CSI 驱动健康状态 → CSIDriver 对象 → 组件日志，高效地定位了根因。

2. **即时缓解**：一旦发现问题，通过补丁 CSIDriver 对象设置正确的 `fsGroupPolicy` 并升级 CSI 驱动迅速解决了问题。

3. **监控集成**：现有监控在节点组刷新后几分钟内就向团队发出告警，最小化了发现时间。

### 关键要点

- **每次 K8s 升级前必须验证存储和 CSI 驱动的兼容性**——而不仅仅是控制平面和节点的兼容性。
- **CSIDriver 对象规格会随时间变化**——始终对照你要升级到的 Kubernetes 版本检查 `kubectl get csidriver -o yaml` 的输出。
- **有状态工作负载需要卷生命周期测试作为升级验证的一部分**——仅验证控制平面组件健康是不够的。
- **CSI 驱动版本与 Kubernetes 版本密切相关**——升级前务必查看 CSI 驱动的发布说明以确认支持的 Kubernetes 版本。

---

## 总结

### 事件时间线

| 时间 | 事件 |
|------|------|
| T-0 | K8s 控制平面从 v1.28 升级到 v1.29 完成 |
| T+5m | 节点组刷新开始——新节点启动 v1.29 kubelet |
| T+10m | 告警触发：有状态 Pod 卡在 Pending 状态 |
| T+15m | 值班工程师开始排查 |
| T+30m | CSI 驱动日志显示节点插件注册失败 |
| T+45m | 发现 CSIDriver 对象缺少 fsGroupPolicy 字段 |
| T+60m | 根因确定：CSI 驱动 v1.20 与 K8s 1.29 不兼容 |
| T+75m | CSI 驱动升级到 v1.30.0 |
| T+80m | 验证 CSIDriver 的 fsGroupPolicy 已设置为 File |
| T+85m | 删除卡住的 Pod——控制器成功重新创建 |
| T+100m | 所有有状态工作负载恢复 |

### 版本兼容性对照表

| Kubernetes 版本 | 所需 CSI 驱动版本 | 所需 node-driver-registrar | fsGroupPolicy |
|--------------------|---------------------------|-------------------------------|---------------|
| v1.27 及更早 | v1.20（可用） | v2.5.x | 不需要 |
| v1.28 | v1.25+ | v2.7+ | 必须设置 |
| v1.29 | v1.30+ | v2.9+ | 必须设置为 File |
| v1.30 | v1.30+ | v2.9+ | 必须设置为 File |

### 关键命令速查

```bash
# 检查 Pod 卷错误
kubectl describe pod <pod名称>

# 检查 PVC 状态
kubectl get pvc -A
kubectl describe pvc <pvc名称>

# 检查 CSIDriver 对象
kubectl get csidriver -o yaml

# 检查 CSI 驱动节点插件日志
kubectl logs -n kube-system -l app=ebs-csi-node csi-driver

# 检查 CSI 驱动控制器日志
kubectl logs -n kube-system -l app=ebs-csi-controller csi-provisioner

# 升级 CSI 驱动
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=v1.30.0"

# 补丁 CSIDriver fsGroupPolicy（紧急）
kubectl patch csidriver ebs.csi.aws.com -p '{"spec":{"fsGroupPolicy":"File"}}'

# 删除所有 Pending Pod 以强制重新创建
kubectl delete pods --field-selector=status.phase=Pending -A

# 检查 kubelet 日志中的卷错误
journalctl -u kubelet | grep -i "volume\|pvc\|csi"

# 检查节点 CSI 插件套接字
ls -la /var/lib/kubelet/plugins/ebs.csi.aws.com/
```

这次事件是一个宝贵的提醒：Kubernetes 存储是一个复杂的子系统，涉及多个移动部件，在集群升级过程中必须仔细管理这些组件之间的版本兼容性。最具有迷惑性的地方在于：现有的 Pod 仍然正常工作——只有新 Pod 创建和滚动更新受到影响——这使得根因在初步排查时不太明显。按照存储栈层次结构（Pod → PVC → CSI 驱动 → CSIDriver 规格 → kubelet 注册）进行系统化排查是定位和解决问题的关键。
