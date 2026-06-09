---
title: "CNI 网络插件故障导致 Pod 间通信异常"
date: 2026-06-10
weight: 100470
slug: "cni-network-plugin-failure"
tags: ["kubernetes", "networking", "cni", "calico", "troubleshooting"]
categories: ["Troubleshooting"]
description: "一次 CNI 网络插件故障实录——集群升级后 Calico 配置与内核模块不匹配导致 Pod 网络连通性全面失效，整个集群的服务间通信彻底中断"
keywords: "cni 网络插件故障, calico 排障, kubernetes pod 网络异常, calico felix 配置, kubernetes cni 无法工作"
draft: false
featured: true
cover:
  image: ""
  caption: "CNI 网络插件故障排查实录"
---

# CNI 网络插件故障导致 Pod 间通信异常

## 常见搜索词

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- cni 网络插件故障 kubernetes pod 无法创建
- network is not ready calico 修复
- failed to create pod sandbox error adding pod to cni
- calico felix ipip 隧道 升级后不可用
- calico vxlan 切换 ipip 排障
- kubernetes 控制平面升级后 cni 不可用
- calicoctl node status 无连接
- kubectl describe pod 网络未就绪

---

## 故障经过

### 环境信息

| 组件 | 版本 / 详情 |
|------|------------|
| Kubernetes | 1.29（从 1.28 升级） |
| CNI 插件 | Calico v3.26 |
| 集群规模 | 30 个工作节点 |
| Pod 数量 | ~500 个 |
|  overlay 模式 | IPIP（IP-in-IP 隧道） |
| 节点操作系统 | Ubuntu 22.04 LTS |

### 时间线

**09:30 UTC** — 平台团队开始对生产集群执行例行 Kubernetes 控制平面升级，从 v1.28 升级到 v1.29。升级按照标准的 `kubeadm upgrade` 流程进行：先升级控制平面节点，再升级工作节点。

**10:15 UTC** — 控制平面升级完成。工作节点升级开始。团队注意到集群中新建的 Pod 全部卡在 `ContainerCreating` 状态。

**10:20 UTC** — `kubectl describe pod` 查看受影响的 Pod 返回 `network is not ready` 和 `failed to create pod sandbox: error adding pod to CNI`。现有 Pod 正常运行，但任何新部署或滚动更新均失败。

**10:25 UTC** — 团队宣布 P0 级事件。所有新部署、HPA 自动扩缩和滚动更新全部被阻塞。

### 症状

- **新 Pod 卡在 ContainerCreating**：升级窗口后创建的所有 Pod 无限期停留在 `ContainerCreating`。
- **现有 Pod 正常工作**：升级前已在运行的 Pod 网络连通性正常，可以互相通信。
- **"network is not ready" 错误**：kubelet 报告节点网络未就绪，无法为新的 Pod 设置网络。
- **CNI 插件错误**：`kubectl describe pod` 显示：
  ```
  Warning  FailedCreatePodSandBox  2m   kubelet  Failed to create pod sandbox: rpc error:
  code = Unknown desc = failed to setup network for sandbox "...":
  plugin type="calico" failed (add): error adding pod to CNI network
  ```
- **集群范围影响**：虽然现有服务保持在线，但所有需要创建新 Pod 的操作——新 Deployment、StatefulSet、DaemonSet、HPA 扩容、CI/CD 流水线——全部被阻断。

---

## 背景

要理解故障原因，我们需要快速回顾 CNI 插件架构在 Kubernetes 中的工作原理。

### CNI 插件架构

当节点上的 kubelet 需要创建 Pod 时，工作流程如下：

```
kubelet → CRI (containerd/CRI-O) → 创建 sandbox → CNI 插件 → 网络配置
```

1. Kubelet 通知容器运行时接口（CRI）创建 Pod sandbox。
2. CRI 运行时调用 CNI 插件（本文中为 Calico）配置 Pod 的网络命名空间。
3. CNI 插件创建虚拟以太网对（veth）、分配 IP 地址、配置路由规则。
4. 如果使用 overlay 网络（IPIP 或 VXLAN），CNI 插件还需要建立封装隧道。

上述任何步骤失败，Pod sandbox 创建都会失败，Pod 停留在 `ContainerCreating`。

### Calico 组件

Calico 由以下几个关键组件组成：

| 组件 | 作用 |
|------|------|
| **Felix** | 运行在每个节点上的核心 Calico 代理。管理网络接口、路由和 ACL 规则。对 Linux 网络栈进行编程。 |
| **BIRD** | BGP 路由反射器客户端。在节点之间分发路由信息。 |
| **confd** | 根据 Calico 数据存储中的数据动态生成 BIRD 配置。 |
| **calicoctl** | 用于管理 Calico 配置和故障排查的命令行工具。 |

### IPIP 与 VXLAN Overlay 对比

IPIP 和 VXLAN 都是 overlay 网络技术，用于在集群底层网络之上封装 Pod 流量：

| 特性 | IPIP | VXLAN |
|------|------|-------|
| 封装方式 | IP-in-IP（协议 4） | MAC-in-UDP |
| 额外开销 | 每包 20 字节 | 每包 50 字节 |
| 内核模块 | `ipip.ko` | `vxlan.ko` |
| 适用场景 | 简单 L3 网络 | L2 网络、云环境 |
| MTU 开销 | 1440（1500 MTU 网络） | 1450（1500 MTU 网络） |

对本故障而言最关键的差异：**IPIP 需要加载 `ipip` 内核模块**，而 VXLAN 需要 `vxlan` 模块。如果这些模块不可用，隧道接口就无法创建，CNI 插件将失败。

---

## 排查过程

我们按系统化的方式逐步缩小排查范围。以下按顺序列出每一个排查步骤。

### 步骤 1：检查 Pod 状态

```bash
kubectl get pods -A | grep ContainerCreating
```

结果显示数十个跨多个命名空间的 Pod 卡在 `ContainerCreating`。现有 Pod 全部为 `Running`。

```bash
kubectl describe pod <卡住的 Pod> -n <命名空间>
```

输出确认：

```
Events:
  Type     Reason                  Age   From               Message
  ----     ------                  ----  ----               -------
  Warning  FailedCreatePodSandBox  10s   kubelet            Failed to create pod sandbox:
    rpc error: code = Unknown desc = failed to setup network for sandbox
    "abc123...": plugin type="calico" failed (add): error adding pod to CNI network
```

**初步判断**：错误明确来自 CNI 插件层。Kubelet 尝试设置 Pod 网络，但 Calico 拒绝或无法完成操作。

### 步骤 2：检查 Kubelet 日志

```bash
journalctl -u kubelet --since "30 min ago" | grep -i cni
```

日志显示重复错误：

```
kubelet[1234]: E0610 10:16:22.123456    1234 cni.go:320] Error validating CNI config:
  error loading config file /etc/cni/net.d/10-calico.conflist: error parsing config:
  unknown plugin type "calico"
```

等等——这个错误具有误导性。Calico 当然是一个有效的插件。真正的问题更深层。更详细的 kubelet 日志显示：

```
kubelet[1234]: E0610 10:17:01.654321    1234 kubelet.go:234] "Pod sandbox setup failed"
  pod="namespace/pod-name" error="failed to setup network for sandbox "...":
  plugin type="calico" failed (add): error adding pod to CNI network
```

Kubelet 正确找到了 Calico CNI 配置，但 Calico 在执行 `ADD` 操作时失败。

### 步骤 3：检查 Calico Pod

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node
```

```
NAME                READY   STATUS    RESTARTS   AGE
calico-node-abc12   1/1     Running   0          12h
calico-node-def34   1/1     Running   0          12h
calico-node-ghi56   1/1     Running   0          12h
...
```

所有 Calico 节点 Pod 均为 `Running` 状态。排除了 Calico 本身崩溃或部署失败的可能性。

### 步骤 4：检查 Calico 节点日志

```bash
kubectl logs -n kube-system calico-node-abc12
```

日志显示了重复模式：

```
2026-06-10 10:15:30.123 [INFO][1234] felix/int_dataplane.go  : Starting dataplane driver: linux
2026-06-10 10:15:30.456 [INFO][1234] felix/int_dataplane.go  : Linux dataplane driver started
2026-06-10 10:15:30.789 [WARN][1234] felix/ipip_mgr.go       : IPIP tunnel setup failed:
  Failed to create IPIP tunnel tunl0: error creating tunnel: operation not supported
2026-06-10 10:15:31.012 [WARN][1234] felix/ipip_mgr.go       : IPIP tunnel setup failed:
  Failed to create IPIP tunnel tunl0: error creating tunnel: operation not supported
```

这是关键的线索。**Felix 反复尝试创建 IPIP 隧道失败**。错误 `operation not supported` 直接指向缺少内核模块，或内核不支持 IPIP 封装。

### 步骤 5：检查节点上的 CNI 配置

```bash
cat /etc/cni/net.d/10-calico.conflist
```

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "kubernetes",
      "nodename": "worker-node-01",
      "mtu": 1440,
      "ipam": {
        "type": "calico-ipam"
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {"portMappings": true}
    }
  ]
}
```

CNI 配置看起来正常。MTU 设置为 1440，这是 1500 MTU 网络下 IPIP 封装的标准值。

### 步骤 6：检查 Calico Felix 配置

```bash
kubectl get FelixConfiguration default -o yaml
```

```yaml
apiVersion: crd.projectcalico.org/v1
kind: FelixConfiguration
metadata:
  name: default
spec:
  ipipEnabled: true
  ipip:
    enabled: true
    mode: Always
  vxlanEnabled: false
  mtu: 1440
```

确认：IPIP 启用并设置为 `Always` 模式，VXLAN 禁用。

### 步骤 7：检查节点上的 IP 隧道

```bash
ip tunnel show
```

如果 IPIP 正常工作，输出应该显示：

```
tunl0: ip/ip  remote any  local any  ttl inherit  nopmtudisc
```

但实际上输出**为空**。`tunl0` 接口在升级后的节点上根本不存在。

```bash
ip link show tunl0
```

```
Device "tunl0" does not exist.
```

确认 IPIP 隧道设备从未被创建。

### 步骤 8：检查内核模块

```bash
lsmod | grep ipip
```

无输出——`ipip` 模块未加载。

```bash
lsmod | grep tunnel
```

显示 `tunnel4` 和 `tunnel6` 存在，但 `ipip` 缺失。

```bash
modprobe ipip
```

命令无报错完成。加载模块后：

```bash
lsmod | grep ipip
```

```
ipip                   16384  0
tunnel4                16384  1 ipip
```

模块已加载。但这只修复了当前节点——重启后会丢失。

### 步骤 9：用 calicoctl 测试节点间连通性

```bash
kubectl exec -n kube-system calico-node-abc12 -- calicoctl node status
```

```
Calico process is running.

IPv4 BGP status
No IPv4 peers found.
```

BGP 状态显示没有对等节点——Calico 与相邻节点之间没有建立 BGP 会话。这是 IPIP 隧道失败的后果：没有可用的隧道，BGP 路由就无法正常交换。

---

## 根因

根因是 **Kubernetes 从 v1.28 升级到 v1.29 过程中引入的内核模块可用性不匹配**。

### 发生了什么

1. **升级过程更新了节点镜像**：当通过 `kubeadm upgrade` 升级工作节点时，流程涉及更新系统包，在某些配置下还会部署新的节点镜像。升级后的节点使用的内核中，`ipip.ko` 模块存在于磁盘上但**默认没有加载**。

2. **Calico 要求 IPIP**：Calico 的 `FelixConfiguration` 指定了 `ipipEnabled: true` 且 `IPIPMode: Always`。这意味着 Calico Felix 会在每个节点启动时尝试创建 IPIP 隧道（`tunl0`）。

3. **IPIP 隧道创建失败**：当 Felix 尝试创建 `tunl0` 接口时，Linux 内核返回 `operation not supported`，因为 `ipip` 内核模块未加载。没有这个模块，内核无法识别 IPIP 封装（协议 4）。

4. **Calico Felix 进入降级状态**：Felix 继续运行并持续报告错误，但无法完成新 Pod 的网络设置。现有 Pod 的网络命名空间未被触及——它们的 veth 对和路由保持不变——这就是为什么现有 Pod 仍然正常工作。

5. **CNI ADD 操作失败**：当 kubelet 要求 Calico 将新 Pod 加入网络时，CNI 插件尝试创建 veth 对并添加依赖于 IPIP 隧道基础设施的路由。由于隧道不可用，整个操作失败，kubelet 将 Pod 留在 `ContainerCreating`。

### 为什么现有 Pod 仍然工作

这是 CNI 故障中经常被误解的一点。现有 Pod 继续运行是因为：

- 它们的网络命名空间在升级前已经创建和配置完毕
- 连接命名空间到主机的 veth 对仍然完好
- 现有 Pod 的 Linux 路由表条目仍然有效
- IPIP 隧道缺失只影响**新的**网络命名空间创建

打个比方：就像一座桥在车辆通过后坍塌了——对面的车仍然可以行驶，但没有新车能够过桥。

### MTU 问题

Calico 配置了 `mtu: 1440`，这在 1500 MTU 网络上对于 IPIP 是正确的（1500 - 20 字节 IPIP 头部 = 1480，但 Calico 还会考虑 Kubernetes 服务封装的额外开销，最终为 1440）。然而，升级后由于 IPIP 无法工作，这个 MTU 设置对新连接已无意义。

---

## 解决方案

我们有两种修复路径：紧急修复（加载 IPIP 模块）和永久配置修复（切换到 VXLAN）。

### 紧急修复：加载 IPIP 内核模块

恢复服务最快的方式是在每个节点上加载缺失的内核模块：

```bash
# 在每个工作节点上执行
modprobe ipip

# 验证加载成功
lsmod | grep ipip
```

然后重启 Calico 使更改生效：

```bash
kubectl rollout restart -n kube-system daemonset calico-node
```

等待 Calico Pod 重启完毕并验证：

```bash
kubectl rollout status -n kube-system daemonset calico-node --timeout=5m

# 验证 IPIP 隧道已创建
kubectl exec -n kube-system calico-node-abc12 -- ip tunnel show
# 预期输出：tunl0: ip/ip  remote any  local any  ttl inherit

# 验证 BGP 对等体
kubectl exec -n kube-system calico-node-abc12 -- calicoctl node status
```

最后，删除所有卡住的 Pod 使其重新创建：

```bash
kubectl delete pods --field-selector=status.phase=Pending -A
```

**重要提醒**：这是临时修复。节点重启会卸载 `ipip` 模块，问题会再次出现。要永久修复，有以下两种选择：
- 将 `ipip` 添加到 `/etc/modules-load.d/`，使其在启动时自动加载，或
- 切换到 VXLAN 模式（推荐）

### 永久修复：切换到 VXLAN 模式

考虑到 VXLAN 在不同内核版本和云环境中的可移植性更好，并且可以避免 IPIP 内核模块依赖问题，我们选择将 Calico overlay 迁移到 VXLAN。

**步骤 1：更新 Calico Installation 资源**

```bash
kubectl edit installation.operator.tigera.io default
```

将封装模式从 IPIP 改为 VXLAN：

```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet  # 原来是: IPIPCrossSubnet
    mtu: 1450  # VXLAN 开销为 50 字节
```

**步骤 2：应用更改**

```bash
kubectl apply -f calico-installation.yaml
```

**步骤 3：重启 Calico**

```bash
kubectl rollout restart -n kube-system daemonset calico-node
```

**步骤 4：重启所有现有工作负载**

这一步是必要的，因为现有 Pod 仍然使用基于 IPIP 的路由和 veth 对。需要完全重启 Pod 才能切换到 VXLAN：

```bash
# 重启所有 Deployment（注意生产流量的影响！）
kubectl get deployments -A -o json | jq -r '.items[] | "kubectl rollout restart deployment -n \(.metadata.namespace) \(.metadata.name)"' | bash
```

**步骤 5：验证切换**

```bash
# 检查 VXLAN 接口是否存在
kubectl exec -n kube-system calico-node-abc12 -- ip link show

# 查找名为 vxlan.calico 的接口

# 验证 BGP 状态
calicoctl node status
```

### 长期预防措施

在解决事件后，我们实施了以下防护措施：

#### 1. CNI 升级前检查脚本

在集群升级流程中加入 CNI 前置检查，在腾空节点之前验证 CNI 插件状态：

```bash
#!/bin/bash
# cni-preflight-check.sh

echo "=== CNI 前置检查 ==="

# 检查内核模块
for mod in ipip vxlan; do
  if lsmod | grep -q "^$mod"; then
    echo "[OK] 内核模块 $mod 已加载"
  else
    echo "[WARN] 内核模块 $mod 未加载"
    modprobe $mod 2>/dev/null && echo "  -> 已成功加载 $mod" || echo "  -> 加载 $mod 失败"
  fi
done

# 检查 Calico 节点状态
kubectl exec -n kube-system daemonset/calico-node -- calicoctl node status 2>/dev/null || echo "[FAIL] calicoctl node status 失败"

# 检查隧道接口
kubectl exec -n kube-system daemonset/calico-node -- ip tunnel show 2>/dev/null

# 检查 Calico Felix 健康状态
kubectl get felixconfiguration default -o yaml | grep -E "ipipEnabled|vxlanEnabled"

echo "=== 检查完成 ==="
```

#### 2. 将 Calico 健康状态纳入节点就绪检查

添加自定义节点就绪检查，验证 Calico 功能是否正常：

```yaml
# 可以作为脚本或自定义控制器添加
# 在 Calico 降级时将节点标记为 NotReady
```

#### 3. 监控 Calico Felix 指标

添加 Prometheus 告警规则，监控 Calico Felix 的重同步状态：

```yaml
# PrometheusRule 示例
groups:
- name: calico.rules
  rules:
  - alert: CalicoFelixResyncStalled
    expr: calico_node_felix_resync_state > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "节点 {{ $labels.node }} 上 Calico Felix 重同步停滞"
```

#### 4. 版本兼容性矩阵

锁定与 K8s 兼容的 Calico 版本并记录对照表：

| Kubernetes | Calico（推荐） |
|-----------|---------------|
| 1.28 | v3.26, v3.27 |
| 1.29 | v3.27, v3.28 |
| 1.30 | v3.28+ |

#### 5. 节点初始化脚本中检查内核模块

在节点初始化/引导脚本中加入内核模块验证（无论使用 `kubeadm`、`Terraform` 还是自定义部署工具）：

```bash
# 在节点引导脚本中
REQUIRED_MODULES=("ipip" "vxlan" "tunnel4" "tunnel6" "br_netfilter" "overlay")
for mod in "${REQUIRED_MODULES[@]}"; do
  modprobe $mod || echo "加载 $mod 失败"
done

# 持久化模块配置，重启后自动加载
cat > /etc/modules-load.d/calico.conf <<EOF
ipip
vxlan
tunnel4
tunnel6
br_netfilter
overlay
EOF
```

#### 6. 记录 IPIP 与 VXLAN 的取舍

| 决策因素 | IPIP | VXLAN |
|---------|------|-------|
| 性能 | 略好（开销更低） | 略差（50 vs 20 字节） |
| 可移植性 | 需要 `ipip.ko`；并非所有云 VM 都可用 | 更广泛支持，在大多数平台可用 |
| 云兼容性 | 部分云平台可能不支持（如 GCP） | 所有主流云平台均支持 |
| MTU 开销 | 20 字节 | 50 字节 |
| 适用场景 | 本地部署、可控环境 | 云环境、多租户、混合环境 |

---

## 经验教训

### 做得好的方面

- **现有 Pod 存活**：由于升级没有重启运行中的 Pod，关键服务在事件期间保持可用。
- **Calico 错误日志明确**：Felix 日志清楚显示 `IPIP tunnel setup failed`，一旦我们找到正确位置，就直指根因。
- **具备回滚能力**：团队备有 `kubeadm upgrade revert` 的操作文档，尽管最终没有使用。

### 做得不足的方面

- **缺少 CNI 前置检查**：升级流程验证了控制平面组件，但没有验证节点镜像更新后 CNI 插件是否能正常运行。
- **假设内核模块默认加载**：我们假设 IPIP 已编译进内核或默认在所有节点镜像上加载。这对旧的基础镜像成立，但对更新后的镜像不成立。
- **缺少 Calico 健康监控**：我们有节点级健康检查（`NodeReady`），但没有 Calico 特定的健康监控。Felix 的降级状态没有触发任何告警。
- **MTU 配置未重新评估**：切换 overlay 模式时需要调整 MTU。我们之前没有验证 VXLAN 作为回退方案的 MTU 配置。

### 关键总结

1. **CNI 是关键的升级依赖项**：在任何集群升级中，将 CNI 插件视为一级组件。在升级工作节点之前明确测试它。
2. **内核模块是基础设施**：不要假设常见的内核模块默认加载。在节点引导脚本中显式配置它们。
3. **监控网络架构**：Calico Felix 指标（特别是 `felix_resync_state` 和 `ipip_tunnel_errors`）应纳入监控体系。
4. **在升级后验证中测试 Pod 创建**：不要只检查节点是否为 `Ready`——验证新 Pod 能否真正创建并正常联网。
5. **为 CNI 层制定回滚计划**：了解主用模式失败时如何快速切换 overlay 模式。

---

## 总结

### 时间线回顾

| 时间 | 事件 |
|------|------|
| 09:30 UTC | K8s 1.28 到 1.29 控制平面升级开始 |
| 10:15 UTC | 工作节点升级完毕；新 Pod 卡在 `ContainerCreating` |
| 10:20 UTC | 宣布事件，开始排查 |
| 10:25 UTC | Calico Felix 日志显示 IPIP 隧道创建失败 |
| 10:35 UTC | 在第一个节点上执行 `modprobe ipip`；重启 Calico |
| 10:45 UTC | 紧急修复推广到所有节点 |
| 11:00 UTC | 删除卡住的 Pod；新 Pod 创建成功 |
| 11:30 UTC | 决定永久迁移到 VXLAN |
| 13:00 UTC | VXLAN 迁移完成并验证 |

### 配置对比

| 参数 | 修复前 (IPIP) | 修复后 (VXLAN) |
|------|--------------|---------------|
| 封装方式 | IPIPAlways | VXLANCrossSubnet |
| MTU | 1440 | 1450 |
| 内核模块 | ipip.ko | vxlan.ko |
| 隧道接口 | tunl0 | vxlan.calico |
| 每包额外开销 | 20 字节 | 50 字节 |

CNI 网络插件是 Kubernetes Pod 通信的基石。这次事件表明，即使是一次"常规"的控制平面升级，如果 CNI 插件的依赖项未经验证，也可能悄然破坏网络层。修复方案——从 IPIP 切换到 VXLAN——不仅解决了当前的故障，还使集群更具可移植性，对未来内核配置差异也更健壮。

---

*本文是 Kubernetes 故障排查系列的一部分。更多真实事件分析，请查看 [故障排查](/categories/troubleshooting/) 归档。*
