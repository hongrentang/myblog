---
title: "磁盘爆满导致 Node NotReady：一次日志轮转失效引发的 Kubernetes 节点故障"
date: 2026-05-19
weight: 100010
slug: "disk-full-node-notready"
tags: ["kubernetes", "storage", "troubleshooting"]
categories: ["存储"]
description: "K8S Node NotReady 根因是磁盘爆满，从误判网络故障到清理日志恢复的排查过程"
keywords: "kubernetes node not ready 解决,k8s 节点磁盘爆满,pod evicted 磁盘压力,节点状态 NotReady"
draft: false
---

# 磁盘爆满导致 Node NotReady

## 问题引导

如果你遇到以下问题，这篇文章可能有帮助：

- kubernetes node not ready 解决
- k8s 节点磁盘爆满
- pod evicted 磁盘压力
- node 状态 NotReady
- 磁盘占用 100% 排查

---

## 故障现象

**环境**：K8S v1.28，containerd 1.7，3 Master + 5 Worker，根盘 100GB。

**时间**：凌晨 4:10，接到告警。

**现象**：Worker 节点 `node-03` 状态变为 `NotReady`，上面的 Pod 全部 `Evicted`。

```bash
kubectl get nodes
NAME      STATUS     ROLES    AGE
node-01   Ready      worker   180d
node-02   Ready      worker   180d
node-03   NotReady   worker   180d    # ← 异常
node-04   Ready      worker   60d
node-05   Ready      worker   60d
```

**影响范围**：node-03 上 30 多个 Pod 下线，包括 2 个核心 API 实例。

---

## 真实场景

凌晨 4 点被告警叫醒。node-03 NotReady，第一反应是 kubelet 挂了或者网络不通。查了下 kubelet 服务还在跑，于是怀疑 Calico 隧道出问题了——之前遇到过类似情况。

准备 drain 节点然后重启 Calico。好在先看了一眼监控，发现了真正的问题。

---

## 排查过程

### 第一步：检查节点状态

```bash
kubectl describe node node-03
```

Conditions 里面有一条关键信息：

```
Conditions:
  Type                 Status  LastHeartbeatTime
  ----                 ------  -----------------
  NetworkUnavailable   False   ...
  MemoryPressure       False   ...
  DiskPressure         True    ...             # ← 磁盘压力
  PIDPressure          False   ...
  Ready                Unknown ...
```

`DiskPressure: True`，说明节点磁盘空间不足。之前怀疑网络完全是误判。

### 第二步：登录节点看磁盘

```bash
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        98G   98G     0G 100% /             # ← 根盘 100%！
/dev/vdb1       500G  200G  300G  40% /data
```

根盘 100%，containerd 和 kubelet 都没法写日志了，节点自然 NotReady。

### 第三步：查什么占了空间

```bash
du -sh /var/log /var/lib/containerd /var/lib/kubelet 2>/dev/null | sort -rh
3.2G    /var/log
45G     /var/lib/containertd     # ← 这是 typo 路径，正解
```

等等，上面的 `containertd` 是手误。重新看实际输出：

```bash
du -sh /var/log /var/lib/containerd /var/lib/kubelet 2>/dev/null | sort -rh
38G     /var/lib/kubelet
25G     /var/log
12G     /var/lib/containerd
```

实际上跑完命令看到的根因是 `/var/log` 占了 25G——里面有大量未被轮转的 Nginx ingress controller 日志。

```bash
ls -lh /var/log/pods/ingress-nginx_ingress-nginx-controller-*/ | head -10
total 12G
-rw-r----- 1 root root 2.0G May 19 02:30 0.log
-rw-r----- 1 root root 2.0G May 19 02:15 1.log
-rw-r----- 1 root root 2.0G May 19 02:00 2.log
```

单个日志文件 2GB，而且没有被 logrotate 切割。

---

## 原因定位

根因是该节点上线时，logrotate 配置没有部署到位，/var/log/pods/ 下的容器日志一直写一直涨，没有自动轮转和压缩。

三个月累积下来，日志最终撑满根盘。

**误判**：认为节点挂了要么是 kubelet 挂了，要么是 Calico 网络断了，完全没看磁盘——因为我潜意识觉得 100GB 根盘不可能被日志打满。

---

## 修复方案

### 紧急恢复（5 分钟）

```bash
# 清理 containerd 无用镜像和容器
crictl rmi --prune

# 清理旧日志
journalctl --vacuum-size=1G

# 手动切割超大日志
find /var/log/pods -name "*.log" -size +1G -exec truncate -s 0 {} \;
```

清理完约 30GB 空间后：

```bash
df -h
/dev/vda1        98G   68G   30G  70% /     # 恢复到 70%
```

几分钟后节点自动恢复 Ready，被驱逐的 Pod 重新调度。

### 临时轮转配置

```bash
cat > /etc/logrotate.d/containers << 'EOF'
/var/log/pods/*/*.log {
    rotate 7
    daily
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 500M
}
EOF
```

### 长期治理

1. **监控磁盘使用率**：Prometheus + Alertmanager 配置磁盘 > 80% 告警
2. **节点压力驱逐阈值**：配置 kubelet 预留空间（系统预留 + kube 预留）

```bash
# /var/lib/kubelet/config.yaml
evictionHard:
  imagefs.available: 15%
  memory.available: 500Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

3. **统一日志采集**：用 Filebeat / Fluentd 把日志转发到外部存储，节点上不长期保留

---

## 避坑记录

- **NotReady 先看 Conditions**：`kubectl describe node` 的 Conditions 里直接写着 `DiskPressure`，比猜网络问题快得多
- **`crictl rmi --prune` 不是必须**：但 containerd 的 image 残留经常有几十 GB，顺手清一下
- **logrotate 不是默认就有的**：容器化环境下，容器日志的轮转依赖 kubelet 的 `containerLogMaxSize` 和 `containerLogMaxFiles` 参数，不是系统 logrotate
- **100GB 根盘不够用**：起码 200GB，或者把 /var/lib 挂独立数据盘

```yaml
# kubelet 容器日志轮转配置
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
containerLogMaxSize: 100Mi
containerLogMaxFiles: 5
```

---

## 总结

排查链路：

```
告警 → node-03 NotReady
  → 误判网络故障（准备重启 Calico）
  → 看到 Conditions: DiskPressure
  → 登录节点，df -h 确认根盘 100%
  → du 找到 /var/log 容器日志 25G
  → 发现 logrotate 未配置
  → crictl + journalctl + truncate 清理
  → 节点恢复 Ready
```

这次故障从接到告警到恢复用了 12 分钟，其中 5 分钟花在误判上。如果第一时间看 node Conditions 而不是凭经验猜，能更快定位。
