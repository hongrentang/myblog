---
title: "磁盘爆满导致节点 NotReady——日志未轮转引发的 Kubernetes 节点宕机"
date: 2026-05-19
weight: 100010
slug: "disk-full-node-notready"
tags: ["kubernetes", "storage", "troubleshooting"]
categories: ["Storage"]
description: "Kubernetes 节点 NotReady 排障实录——从误判 Calico 到发现根因是根盘爆满"
keywords: "kubernetes node not ready disk pressure, k8s 节点磁盘满, pod 被驱逐 磁盘压力, 容器日志轮转"
draft: false
featured: true
cover:
  image: "/images/disk-full-banner.svg"
  caption: "磁盘爆满——Node NotReady 排障"
---

# 磁盘爆满导致节点 NotReady

## 常见搜索词

- kubernetes node not ready 修复
- k8s 节点磁盘满根因分析
- pod 因磁盘压力被驱逐
- 容器日志轮转 kubernetes
- crictl prune 清理未使用镜像

---

## 故障经过

**环境**: K8S v1.28, containerd 1.7, 3 Master + 5 Worker, 根盘 100GB。

**时间**: 凌晨 04:10，监控告警触发。

**症状**: Worker 节点 `node-03` 变为 `NotReady`，其上所有 Pod 被 `Evicted`。

```bash
kubectl get nodes
NAME      STATUS     ROLES    AGE
node-01   Ready      worker   180d
node-02   Ready      worker   180d
node-03   NotReady   worker   180d    # ← 出问题了
node-04   Ready      worker   60d
node-05   Ready      worker   60d
```

**影响**: node-03 上 30+ 个 Pod 下线，包含 2 个关键 API 实例。

---

## 背景

凌晨 4 点，BP 机响了。node-03 NotReady。我的第一反应是 kubelet 或 Calico 出了问题——之前我们遇到过 Calico 隧道抖动。我正准备 drain 节点重启 Calico。

还好先看了一眼监控面板。

---

## 排查过程

### 第一步：检查节点状态

```bash
kubectl describe node node-03
```

一行信息引起了注意：

```
Conditions:
  Type                 Status  LastHeartbeatTime
  ----                 ------  -----------------
  NetworkUnavailable   False   ...
  MemoryPressure       False   ...
  DiskPressure         True    ...             # ← 问题在这里
  PIDPressure          False   ...
  Ready                Unknown ...
```

`DiskPressure: True`。节点磁盘空间不足。Calico 的猜测完全错误。

### 第二步：SSH 进节点检查磁盘

```bash
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        98G   98G     0G 100% /             # ← 根盘满了！
/dev/vdb1       500G  200G  300G  40% /data
```

根盘使用率 100%。kubelet 和 containerd 无法写入日志，节点宣告自身不健康。

### 第三步：定位罪魁祸首

```bash
du -sh /var/log /var/lib/containerd /var/lib/kubelet 2>/dev/null | sort -rh
38G     /var/lib/kubelet
25G     /var/log
12G     /var/lib/containerd
```

`/var/log` 占了 25GB。继续深挖：

```bash
ls -lh /var/log/pods/ingress-nginx_ingress-nginx-controller-*/ | head -10
total 12G
-rw-r----- 1 root root 2.0G May 19 02:30 0.log
-rw-r----- 1 root root 2.0G May 19 02:15 1.log
-rw-r----- 1 root root 2.0G May 19 02:00 2.log
```

单个 2GB 的日志文件，完全没有轮转。Nginx ingress controller 的日志已经连续写了几个月。

---

## 根因

节点上线时，从未为 `/var/log/pods/` 下的容器日志配置 logrotate。三个月下来，ingress controller 的日志累积到撑爆了根盘。

**误判原因**：我直接冲着"kubelet 或 Calico 挂了"去排查，因为上次故障是它们导致的。我根本没看磁盘用量——以为 100GB 对系统日志来说绰绰有余。

---

## 解决方案

### 紧急恢复（5 分钟）

```bash
# 清理未使用的容器镜像
crictl rmi --prune

# 清理 systemd journal
journalctl --vacuum-size=1G

# 截断过大的 Pod 日志
find /var/log/pods -name "*.log" -size +1G -exec truncate -s 0 {} \;
```

释放约 30GB 后：

```bash
df -h
/dev/vda1        98G   68G   30G  70% /     # 回到 70%
```

几分钟内节点恢复 Ready，被驱逐的 Pod 重新调度。

### 临时修复

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

### 长期预防

1. **监控磁盘用量**：Prometheus + Alertmanager 设 80% 告警阈值
2. **Kubelet 驱逐阈值**：为系统和 kubelet 操作预留空间

```bash
# /var/lib/kubelet/config.yaml
evictionHard:
  imagefs.available: 15%
  memory.available: 500Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

3. **外部日志转发**：使用 Filebeat / Fluentd 将日志转发到集中存储，不要在节点本地长期保留

---

## 经验教训

- **NotReady → 先看 Conditions**：`kubectl describe node` 直接告诉你 `DiskPressure`，比瞎猜快得多
- **`crictl rmi --prune` 是救命稻草**：containerd 常积累几十 GB 的 dangling 镜像
- **Logrotate 不会自动作用于容器日志**：Kubelet 有自己的 `containerLogMaxSize` 和 `containerLogMaxFiles` 配置——用它们，而不是依赖系统 logrotate

```yaml
# Kubelet 容器日志轮转配置
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
containerLogMaxSize: 100Mi
containerLogMaxFiles: 5
```

- **100GB 根盘不够用**：至少 200GB，或者把 /var/lib 挂载到独立数据盘

---

## 总结

排查链路：

```
告警 → node-03 NotReady
  → 误判 Calico（正准备重启）
  → 看到 Conditions: DiskPressure
  → SSH 登入，df -h 确认根盘 100%
  → du 发现 /var/log 占 25GB
  → Logrotate 从未配置
  → crictl + journalctl + truncate 清理
  → 节点恢复 Ready
```

总耗时：12 分钟。其中 5 分钟浪费在错误假设上。先检查节点 Conditions——它们存在是有原因的。
