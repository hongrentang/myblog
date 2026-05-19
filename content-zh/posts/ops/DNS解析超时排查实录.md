---
title: "DNS 解析超时导致微服务调用异常"
date: 2026-05-19
weight: 100020
slug: "dns-resolution-timeout-troubleshooting"
tags: ["dns", "网络", "coredns"]
categories: ["网络"]
description: "完整复盘一次因 CoreDNS 上游超时导致全站接口延迟飙升的故障 — 从误判应用层到定位 DNS 链路"
keywords: "dns解析超时, coredns超时, kubernetes dns故障, dns排查, ndots配置"
draft: false
featured: true
cover:
  image: "/images/dns-timeout-banner.svg"
  caption: "DNS 解析超时排查实录"
---

# DNS 解析超时导致微服务调用异常

## 常见搜索词

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- dns解析超时 kubernetes
- coredns 5s timeout
- 微服务调用偶尔失败 dns
- ndots 5 导致 dns 查询过多
- kubectl 检查 dns 解析

---

## 故障现象

**环境**：K8S v1.28, Calico CNI, CoreDNS 1.11, 200+ 微服务。

**时间**：周三 14:30，低峰期。

**症状**：监控告警——订单服务的 P99 延迟从 50ms 飙升至 3.2s，大量请求返回 503。部分上游服务返回 `connection refused`。

```bash
# 告警信息
HTTP P99 Latency: 3240ms (threshold: 500ms)
Error Rate: 12.7% (threshold: 1%)
```

**影响**：用户下单需等待 3-5 秒，部分订单提交失败。

---

## 误判阶段

### 第一反应：应用层问题

看到 P99 3.2s，第一反应是上游服务负载过高或出现慢查询。我直接登录了订单 Pod 查看日志：

```bash
kubectl logs -n production order-service-7b8f9d4c5f-abc12 --tail 100
```

日志显示大量 `connection refused` 和 `no such host` 错误，但目标服务明明运行正常，Endpoint 也正常。这说明——不是服务挂了，而是**找不到服务**。

### 怀疑方向转向 DNS

```
2026-05-19T14:25:18.003Z ERROR Get "http://user-service.production.svc.cluster.local/api/check": dial tcp: lookup user-service.production.svc.cluster.local on 10.96.0.10:53: i/o timeout
```

关键信息：**i/o timeout**，目标地址 `10.96.0.10:53`——这是 CoreDNS 的 Service IP。

---

## 排查过程

### Step 1: 验证 Pod 内 DNS 解析

```bash
kubectl exec -n production order-service-7b8f9d4c5f-abc12 -- nslookup user-service.production.svc.cluster.local
```

结果：

```
Server:    10.96.0.10
Address:   10.96.0.10:53

** server can't find user-service.production.svc.cluster.local: NXDOMAIN
```

直接查完整 FQDN 返回 NXDOMAIN，但有时又正常——典型的**间歇性超时**表现。

### Step 2: 查看 Pod DNS 配置

```bash
kubectl exec -n production order-service-7b8f9d4c5f-abc12 -- cat /etc/resolv.conf
```

```
nameserver 10.96.0.10
search production.svc.cluster.local svc.cluster.local cluster.local
ndots: 5
```

`ndots: 5` 是关键。当查询的域名中点的数量少于 5 时，Kubernetes 会依次拼接 search 域名进行查询。例如查询 `user-service`（0 个点），实际会产生 4 次 DNS 查询：

1. `user-service.production.svc.cluster.local.`
2. `user-service.svc.cluster.local.`
3. `user-service.cluster.local.`
4. `user-service.`（绝对路径）

每次查询都走一遍完整的 DNS 链路，如果上游超时，总延迟会成倍放大。

### Step 3: 检查 CoreDNS 状态

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7d6d9b7f4d-8k9m2  1/1     Running   0          12d
coredns-7d6d9b7f4d-x3p1q  1/1     Running   0          12d
```

Pod 正常，没有重启。继续查日志：

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail 100
```

```
[ERROR] plugin/errors: 2 user-service.production.svc.cluster.local. A: dial tcp 8.8.8.8:53: i/o timeout
[ERROR] plugin/errors: 2 user-service.svc.cluster.local. A: dial tcp 8.8.8.8:53: i/o timeout
[ERROR] plugin/errors: 2 user-service.cluster.local. A: dial tcp 8.8.8.8:53: i/o timeout
```

CoreDNS 配置了上游转发到 `8.8.8.8` 和 `223.5.5.5`，但到 `8.8.8.8` 的连接间歇性超时。CoreDNS 默认上游超时 5s，加上 ndots 导致多次查询叠加，总延迟 = 5s × 4 次 ≈ 20s。

### Step 4: 确认是 CoreDNS 上游问题

查看 CoreDNS ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        forward . 8.8.8.8 223.5.5.5 {
          max_concurrent 1000
        }
        cache 30
        errors
    }
```

上游 DNS 有两个：`8.8.8.8` 和 `223.5.5.5`。问题在于 `8.8.8.8` 从内网访问存在网络策略阻断，导致 TCP 连接超时。`forward` 插件是串行尝试的，当一个上游挂了，查询延迟大幅增加。

---

## 根因

1. CoreDNS 配置了 `8.8.8.8` 作为上游 DNS，但**内网防火墙规则变更后**，K8S 节点到 `8.8.8.8` 的 TCP/53 被阻断
2. CoreDNS `forward` 插件在超时（默认 5s）后才 fallback 到 `223.5.5.5`
3. Pod 内 `ndots: 5` 导致每个短域名查询产生 3-4 次 DNS 查询，每次等待 5s 超时
4. 应用层没有 DNS 缓存机制，每次请求都触发 DNS 解析

**最大误判**：前 10 分钟我一直盯着应用日志和调用链，认为是某个服务慢查询导致连锁超时。实际上问题在 DNS 层，所有表象——503、延迟飙升、connection refused——都是 DNS 超时的衍生症状。

---

## 恢复过程

### 临时恢复（2 分钟）

移除故障的 `8.8.8.8` 上游，仅保留 `223.5.5.5`：

```bash
kubectl edit configmap coredns -n kube-system
```

修改为：

```yaml
forward . 223.5.5.5 {
  max_concurrent 1000
}
```

重启 CoreDNS：

```bash
kubectl rollout restart -n kube-system deployment/coredns
```

改动生效后，查询延迟从 5s 降到 20ms。P99 延迟在 1 分钟内恢复到 60ms。

---

## 长期修复

### 1. CoreDNS 配置优化

只配置可达的上游，并增加超时和重试策略：

```yaml
forward . 223.5.5.5 114.114.114.114 {
  max_concurrent 1000
  expire 10s
  policy sequential
  health_check 5s
}
```

### 2. 增加缓存时间

将 CoreDNS 缓存从 30s 提高到 60s，降低上游查询频率：

```yaml
cache 60
```

### 3. 应用层增加 DNS 缓存

在服务中配置 DNS 缓存（Java JVM 默认缓存策略）：

```bash
# JVM 参数
-Dsun.net.inetaddr.ttl=60
-Dsun.net.inetaddr.negative.ttl=10
```

### 4. 监控 CoreDNS 上游健康

```yaml
# CoreDNS 配置健康检查
forward . 223.5.5.5 114.114.114.114 {
  health_check 5s
}
```

Node 级别健康检查：

```bash
# 部署 coredns 健康检查 daemonset
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dns-checker
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: dns-checker
  template:
    metadata:
      labels:
        app: dns-checker
    spec:
      containers:
      - name: checker
        image: busybox:1.36
        command:
        - sh
        - -c
        - "while true; do nslookup kubernetes.default.svc.cluster.local >/dev/null 2>&1; sleep 10; done"
EOF
```

---

## 复现步骤

1. 在 CoreDNS ConfigMap 中添加一个不可达的上游 DNS
2. 重启 CoreDNS
3. 进入任意 Pod 执行 `nslookup some-service`，观察 5s+ 延迟
4. 检查 CoreDNS 日志确认上游超时

---

## 排查命令速查

```bash
# 验证 Pod DNS
kubectl exec -it <pod> -- nslookup <service.namespace.svc.cluster.local>

# 查看 DNS 配置
kubectl exec -it <pod> -- cat /etc/resolv.conf

# 检查 CoreDNS 日志
kubectl logs -n kube-system -l k8s-app=kube-dns --tail 50

# 直接测试 CoreDNS Service
kubectl run -it --rm dns-test --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local 10.96.0.10

# 查看 CoreDNS 配置
kubectl get configmap coredns -n kube-system -o yaml

# 重启 CoreDNS
kubectl rollout restart -n kube-system deployment/coredns
```

---

## 总结

排查链路：

```
P99 飙升 → 误判应用慢查询
  → 发现 connection refused + no such host
  → 测试 DNS 解析超时
  → 查看 resolv.conf（ndots: 5）
  → 检查 CoreDNS 日志（上游 8.8.8.8 超时）
  → 移除非可达上游 → 恢复正常
```

总耗时：从告警到恢复 12 分钟。前 10 分钟走了完全错误的方向。

**教训**：延迟飙升 + connection refused，优先排查 DNS，而不是应用层。DNS 是整个集群的命脉，它出问题了所有服务都会表现出奇怪的症状。
