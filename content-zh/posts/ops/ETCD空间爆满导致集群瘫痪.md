---
title: "ETCD 空间爆满导致集群瘫痪——kubectl 完全不可用的 90 分钟"
date: 2026-05-20
weight: 100080
slug: "etcd-space-full-cluster-outage"
tags: ["kubernetes", "etcd", "troubleshooting"]
categories: ["K8S"]
description: "完整复盘一次 ETCD 空间爆满导致 K8S 集群瘫痪的故障——从误判 API Server 到定位 etcd NOSPACE alarm"
keywords: "etcd空间满, etcd quota exceeded, kubernetes集群不可用, etcd NOSPACE, etcd compaction, kubectl无法使用"
draft: false
featured: true
cover:
  image: "/images/etcd-space-full-banner.svg"
  caption: "ETCD 空间爆满导致集群瘫痪"
---

# ETCD 空间爆满导致集群瘫痪——kubectl 完全不可用的 90 分钟

## 背景

周三下午，我正在排查另一个问题的日志，突然收到一大堆告警——不是磁盘告警、不是 Pod 告警，而是 **K8S API Server 不可用**。

```bash
kubectl get nodes
```

```
E0520 14:32:15.123456   12345 memcache.go:265] couldn't get current server API group list
E0520 14:32:15.123789   12345 memcache.go:265] Get "https://api.k8s-cluster.example.com:6443/api?timeout=32s": dial tcp: connect: connection refused
```

kubectl 连不上 API Server。再试：

```bash
kubectl get pods -A
```

```
The connection to the server api.k8s-cluster.example.com was refused - did you specify the right host or port?
```

CI/CD 全挂，新服务部署不了。更糟的是，有些核心服务的 liveness probe 走的是 kubelet 自带的 HTTP 检查还好，但那些依赖 API Server 做服务发现的组件已经开始报错了。

## 排查过程

### 错误尝试 1：以为 API Server 挂了

kubectl 连不上，第一反应——"API Server 挂了，重启一下"。

我有节点 SSH 权限，直接 SSH 到 master 节点看 kube-apiserver 容器状态：

```bash
ssh master-01
crictl ps | grep kube-apiserver
```

```
f3a2b1c0d1e2   3 minutes ago   Running   kube-apiserver
```

Running？那为什么连不上？

```bash
# 查 API Server 日志
crictl logs --tail 50 f3a2b1c0d1e2
```

```
W0520 14:28:01.123456   12345 dispatcher.go:201] Failed to update etcd members list: clientv3: etcdserver: request timed out
W0520 14:28:11.123456   12345 dispatcher.go:201] Failed to update etcd members list: clientv3: etcdserver: request timed out
E0520 14:28:21.123456   12345 dispatcher.go:201] Skipping update of etcd members list (retried 3 times): clientv3: etcdserver: request timed out
```

API Server 在狂报 etcd 超时。那就不是 API Server 的问题，是 etcd 出事了。

**踩坑点**：kubectl 连不上 API Server，第一反应就重启 API Server 就没意义。API Server 本身是无状态的——它不是挂了，而是 etcd 这个后端不可用。重启了也一样会超时。还好我当时没真的重启，先看了眼日志。

### 错误尝试 2：以为 etcd 节点宕了

etcd 超时，那 etcd 是不是挂了？直接查 etcd 集群状态：

```bash
# 在 master 节点上查 etcd 容器
crictl ps | grep etcd
```

```
d4e5f6a7b8c9   3 hours ago   Running   etcd
```

也在 Running。看看日志：

```bash
crictl logs --tail 50 d4e5f6a7b8c9
```

```
{"level":"warn","msg":"rejected connection","remote-addr":"10.0.0.1:43876","error":"etcdserver: mvcc: database space exceeded"}
{"level":"warn","msg":"rejected connection","remote-addr":"10.0.0.2:43877","error":"etcdserver: mvcc: database space exceeded"}
{"level":"warn","msg":"rejected connection","remote-addr":"10.0.0.3:43878","error":"etcdserver: mvcc: database space exceeded"}
```

**"database space exceeded"**——etcd 的空间配额打满了。这不是 etcd 挂了，是 etcd 处于保护模式，拒绝了所有写操作（读操作还可以）。

```bash
# 查看 etcd 当前存储状态
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

```
+------------------+----------------+---------+---------+-----------+-----------+------------+
|    ENDPOINT      |    DB SIZE     |  IN USE  |  RAFT TERM | RAFT INDEX | IS LEADER | ERR MSGS  |
+------------------+----------------+---------+---------+-----------+-----------+------------+
| https://127.0.0.1:2379 | 2.1 GB | 2.0 GB | 123456 | 987654 | true | "NOSPACE" |
+------------------+----------------+---------+---------+-----------+-----------+------------+
```

DB SIZE 2.1 GB。默认的 `--quota-backend-bytes` 是 2 GB（即 2.147 GB）。打满了。

```bash
ETCDCTL_API=3 etcdctl ... alarm list
```

```
memberID: 1234567890 alarm: NOSPACE
```

明确告诉你：**NOSPACE**。这是一个保护机制——etcd 检测到 DB 文件超过配额后，主动拒绝所有写操作，防止 etcd 把磁盘撑爆导致整个集群不可逆地损坏。

### 错误尝试 3：直接压缩，发现没用

知道是空间满了，我的第一反应是压缩。etcd 有 compact 命令：

```bash
# 获取当前 revision
ETCDCTL_API=3 etcdctl ... endpoint status
```

输出的 RAFT INDEX 是 987654，但 etcd 的 compaction 需要的是 **revision**，不是 index。

```bash
# 正确的做法：先拿 revision
ETCDCTL_API=3 etcdctl ... endpoint status -w json | jq -r '.[].Status.revision'
```

```
8234567
```

然后压缩：

```bash
# 压缩所有旧版本数据
ETCDCTL_API=3 etcdctl ... compact 8234567
```

```
compacted revision 8234567
```

压缩成功了。然后我做了第二个错误操作——**只 compact 却没有 defrag**。

我兴冲冲地试 kubectl：

```bash
kubectl get nodes
```

还是 connection refused。

**踩坑点**：很多人跟我一样，以为 compact 完就自动释放空间了。不是的。compact 只是把旧版本数据标记为可回收，**空间还在 etcd 的 DB 文件里，需要主动 defrag 才能释放**。我当时不知道 defrag 这回事，又浪费了 15 分钟。

## 根因分析

```bash
# 看看 etcd 里什么数据占最多
ETCDCTL_API=3 etcdctl ... endpoint status -w json | jq '.'
```

```
{
  "Endpoint": "https://127.0.0.1:2379",
  "Status": {
    "dbSize": 2147483648,
    "dbSizeInUse": 1234567890,
    ...
  }
}
```

两个关键指标：
- **dbSize**：2.14 GB（配额打满）
- **dbSizeInUse**：实际有效数据只有 1.2 GB 左右

差距接近 1 GB——**全是 etcd 没有清理的历史版本数据**。

etcd 是一个 MVCC 数据库，每次对同一个 key 的修改都会产生一个新版本，旧版本不会自动删除。如果集群中频繁变更的资源很多（ConfigMap 更新、Endpoint 变动、Lease 续期），历史版本就会迅速累积。

具体到我们的集群：

```bash
# 查哪些资源变更最频繁
ETCDCTL_API=3 etcdctl ... get / --prefix --keys-only | awk -F '/' '{print $2}' | sort | uniq -c | sort -rn | head -5
```

```
   2345678 /registry/events
    987654 /registry/endpointslices
    456789 /registry/configmaps
    234567 /registry/pods
    123456 /registry/nodes
```

events 占了将近一半。**事件的 TTL 机制是 kubelet 在做，但删除后 etcd 里只是标记删除，旧版本还留着**。再加上 endpointslices 的频繁更新，历史版本数据迅速撑爆了 2 GB 配额。

```
- 直接原因：etcd DB 文件达到 2 GB `quota-backend-bytes` 上限，触发 NOSPACE 保护模式
- 根本原因：没有配置 etcd 自动压缩和碎片整理。历史版本数据（尤其是 events 和 endpointslices）持续累积未被清理
- 触发因素：集群运行时间过长（一年多），从未做过 etcd 维护
```

## 解决方案

### 紧急恢复：compact + defrag

知道了问题，修复其实就两步：

**第一步：压缩历史版本**

```bash
# 1. 获取当前 revision
REV=$(ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w json | jq -r '.[].Status.revision')
echo "Current revision: $REV"

# 2. 压缩到当前 revision（保留最新版本，删除所有历史版本）
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact "$REV"
```

**第二步：整理碎片（释放磁盘空间）**

```bash
# 3. 对每个 etcd 成员执行碎片整理
# 先对 leader 做
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag

# 然后对其他成员做（如果有多个 etcd 节点）
ETCDCTL_API=3 etcdctl \
  --endpoints=https://10.0.0.2:2379 \
  ... \
  defrag
```

**第三步：解除 alarm**

```bash
# 4. 释放空间后取消 NOSPACE alarm
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  alarm disarm

# 5. 验证 alarm 已消除
ETCDCTL_API=3 etcdctl ... alarm list
```

**第四步：验证恢复**

```bash
# 确认 etcd 状态恢复正常
ETCDCTL_API=3 etcdctl ... endpoint status --write-out=table
```

```
+------------------+----------------+---------+---------+-----------+-----------+----------+
|    ENDPOINT      |    DB SIZE     |  IN USE  |  RAFT TERM | RAFT INDEX | IS LEADER | ERR MSGS |
+------------------+----------------+---------+---------+-----------+-----------+----------+
| https://127.0.0.1:2379 | 1.1 GB | 1.1 GB | 123456 | 987789 | true | "" |
+------------------+----------------+---------+---------+-----------+-----------+----------+
```

DB SIZE 从 2.1 GB 降到了 1.1 GB，ERR MSGS 为空。

```bash
# 试 kubectl
kubectl get nodes
```

```
NAME       STATUS   ROLES                  AGE   VERSION
master-01  Ready    control-plane,master   456d v1.28.0
worker-01  Ready    <none>                 456d v1.28.0
worker-02  Ready    <none>                 456d v1.28.0
```

✅ **恢复验证**：
- `kubectl get nodes` 正常返回
- `etcdctl alarm list` 为空
- API Server 日志不再报 etcd 超时
- CI/CD 恢复

### 长期修复

```bash
# 1. 开启自动压缩（按版本压缩：每 1000 个版本压缩一次）
# 编辑 etcd 启动参数或 static Pod 定义
# /etc/kubernetes/manifests/etcd.yaml
```

```yaml
spec:
  containers:
  - command:
    - etcd
    - --auto-compaction-mode=revision
    - --auto-compaction-retention=1000
    # 或者按时间压缩（每 8 小时压缩一次）
    # - --auto-compaction-mode=periodic
    # - --auto-compaction-retention=8h
```

```bash
# 2. 增加 etcd DB 配额（治标，给更多缓冲空间）
# --quota-backend-bytes=8589934592  # 8 GB
```

但是光有 auto-compaction 还不够——compact 只标记不释放空间。还需要：

```bash
# 3. 定时 defrag（cronjob，每周跑一次）
# 用 CronJob 在集群内执行 defrag
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-defrag
  namespace: kube-system
spec:
  schedule: "0 3 * * 0"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-defrag
            image: bitnami/etcd:3.5
            command:
            - etcdctl
            - --endpoints=https://etcd:2379
            - --cacert=/etc/etcd/ca.crt
            - --cert=/etc/etcd/server.crt
            - --key=/etc/etcd/server.key
            - defrag
            volumeMounts:
            - mountPath: /etc/etcd
              name: etcd-certs
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
EOF
```

**auto-compaction 解决"为什么堆积"**，**定时 defrag 解决"为什么不释放"**，两个缺一不可。

```bash
# 4. 减小 events 保留时间
# kube-apiserver 启动参数
# --event-ttl=1h  # 默认 1h，可以根据需要调小
```

**对比**：

| 措施 | 作用 | 为什么能防复发 |
|------|------|--------------|
| auto-compaction | 自动压缩旧版本 | 从源头减少历史数据堆积 |
| 定时 defrag | 释放 DB 空间 | compact 只标记，defrag 才真正释放 |
| 监控 etcd DB 大小 | 提前预警 | 配额用到 80% 就告警，不用等到 NOSPACE |

## 教训总结

1. **kubectl 连不上 = API Server ≠ etcd 挂了**。排查链路是 kubectl → API Server → etcd。API Server 报 etcd 超时时，直接查 etcd 比重启 API Server 有用得多。重启 API Server 解决不了 etcd 的问题。

2. **"database space exceeded" 是保护机制而不是故障。** etcd 在 DB 文件超过配额后会主动拒绝写操作，防止写爆磁盘导致数据损坏。这不是 bug，是 feature——但如果你没配置 compact 和 defrag，迟早会触发它。

3. **compact ≠ defrag，很多人只做了第一步。** compact 只标记历史版本为可回收，空间还在 DB 文件里。defrag 才真正把空间释放给操作系统。从 2GB 释放到 1.1GB 那 900MB 差异就是新旧对比。

4. **events 是 etcd 空间的隐形杀手。** 一个活跃集群每天的 events 变更量是其他资源的好几倍。虽然 event 有 TTL，但 etcd 的 MVCC 机制不会自动清理旧版本。要么开 auto-compaction，要么减小 event TTL。

5. **运行一年以上的集群一定要检查 etcd 维护配置。** 很多集群搭建时用了默认配置，默认的 auto-compaction 是关闭的。等到 NOSPACE 告警时已经是 emergency 了。配一个定时 defrag 的 CronJob，每周跑一次，成本极低但能避免一次重大故障。
