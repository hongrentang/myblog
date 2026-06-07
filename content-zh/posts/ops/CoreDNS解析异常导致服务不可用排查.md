---
title: "CoreDNS 解析异常导致服务不可用排查"
date: 2026-06-07
weight: 100390
slug: "coredns-resolution-failure"
tags: ["coredns", "kubernetes", "troubleshooting", "dns", "network"]
categories: ["Troubleshooting"]
description: "一次 CoreDNS 解析异常故障分析——ndots 配置不当与 CoreDNS Pod 资源耗尽如何导致整个集群 DNS 解析大面积失败"
keywords: "coredns 排查, kubernetes dns 解析失败, coredns 资源限制, ndots kubernetes, dns 超时 kubernetes pod"
draft: false
featured: true
cover:
  image: ""
  caption: "CoreDNS 解析异常排查实录"
---

# CoreDNS 解析异常导致服务不可用排查

## 常见搜索词

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- coredns 解析异常 kubernetes
- kubernetes dns 间歇性失败 部分pod正常 部分异常
- coredns 未设置资源限制被限流
- ndots 5 导致 dns 查询过多 搜索域膨胀
- coredns 自动扩缩容 副本数不足 大集群
- kubectl 排查 coredns dns 解析

---

## 故障经过

**环境**：K8S v1.28, Calico CNI, CoreDNS 1.10.1, 50 个工作节点, 500+ 服务, 2000+ Pod。

**时间**：周二上午 10:30，业务早高峰。

**症状**：监控面板一片飘红——数十个服务同时报告 DNS 解析失败。应用在连接其他服务时抛出 `Name or service not known` 错误。同一个 Deployment 内的部分 Pod 可以正常解析 DNS，而另一部分却不行。故障呈间歇性，且随着负载增加愈发严重。

```bash
# 前 5 分钟触发的告警
Service: order-svc  |  Error: dial tcp: lookup payment-svc on 10.96.0.10:53: no such host
Service: api-gateway |  Error: dial tcp: lookup user-svc on 10.96.0.10:53: i/o timeout
Service: notification | Error: dial tcp: lookup alertmanager on 10.96.0.10:53: no such host
```

**影响**：早高峰期间，下单和支付流程完全中断约 12 分钟。约 15% 的服务间调用失败，导致三个核心业务域出现级联局部宕机。10:33 确认为影响收入的 P0 级事故。

---

## 背景

### CoreDNS 在 Kubernetes 中的架构

CoreDNS 自 K8s v1.13 起成为默认 DNS 解析器。它以 Deployment 形式运行在 `kube-system` 命名空间中，通过 `kube-dns` ClusterIP 服务（默认 `10.96.0.10`）对外暴露。集群中的每个 Pod 都会通过 kubelet 注入的 `/etc/resolv.conf` 将此 IP 作为 nameserver。

```
Pod → /etc/resolv.conf (nameserver 10.96.0.10) → kube-dns Service → CoreDNS Pod → 上游 DNS
```

CoreDNS 承担两类解析任务：
- **集群 DNS**：解析 Kubernetes Service 名称（`service.namespace.svc.cluster.local`）
- **外部 DNS**：将非集群查询转发到上游解析器（如节点的 `/etc/resolv.conf`）

### Pod 中的 DNS 解析——/etc/resolv.conf

Pod 启动时，kubelet 会生成具有特定结构的 `/etc/resolv.conf`：

```
nameserver 10.96.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local <节点搜索域>
options ndots:5
```

这里的 `search` 行和 `ndots` 选项是本故事的关键。它们控制**短名称**（非完全限定域名，即不以点结尾的名称）的 DNS 解析行为。

### 什么是 ndots？

`ndots` 选项告诉 DNS 解析器："如果待解析的名称中点数量少于 N，则先尝试依次附加各个搜索域进行查询。"使用 Kubernetes 默认的 `ndots:5` 时：

- 像 `payment-svc`（0 个点）这样的短名称会触发**多次 DNS 查询**：
  1. `payment-svc.<namespace>.svc.cluster.local.`
  2. `payment-svc.svc.cluster.local.`
  3. `payment-svc.cluster.local.`
  4. `payment-svc.<节点搜索域-1>.`
  5. `payment-svc.<节点搜索域-2>.`
  6. `payment-svc.`（绝对查询——仅当以上全部失败时）

一次短名称查询产生 **6 次 DNS 请求**。再乘以数百个服务和每秒数千次的调用量，后果可想而知。

---

## 排查过程

### 第一步：从调试 Pod 测试基本 DNS 解析

每位 K8s 工程师的第一反应——启动调试 Pod 测试基础解析：

```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local
```

结果——**间歇性**：

```
# 第一次——成功
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

# 30 秒后第二次——失败
;; connection timed out; no servers could be reached
```

确认问题出在 DNS。但为什么是间歇性的？

### 第二步：检查 CoreDNS Pod 状态

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7d6f9b7c8f-ab12c   1/1     Running   0          12d
coredns-7d6f9b7c8f-xy34d   1/1     Running   3          12d
```

两个副本。其中一个已经重启了 3 次，这很可疑。

### 第三步：查看 CoreDNS 日志

```bash
kubectl logs -n kube-system coredns-7d6f9b7c8f-ab12c --tail 100
```

```
[INFO] 10.244.1.15:33112 - 60130 "A IN payment-svc.production.svc.cluster.local. udp 72 false 512" NXDOMAIN qr,rd,ra 138 0
[INFO] 10.244.1.15:33112 - 60131 "A IN payment-svc.production.svc.cluster. udp 60 false 512" NXDOMAIN qr,rd,ra 126 0
[INFO] 10.244.1.15:33112 - 60132 "A IN payment-svc.production.svc. udp 48 false 512" NXDOMAIN qr,rd,ra 114 0
[INFO] 10.244.1.15:33112 - 60133 "A IN payment-svc.production. udp 38 false 512" NXDOMAIN qr,rd,ra 104 0
[WARNING] plugin/forward: connect to upstream: connection refused
[ERROR] plugin/errors: 2 5992878037655751999.5355688120039757079. Hinfo: unreachable backend: read udp 10.244.1.10:42135->10.0.0.1:53: i/o timeout
```

可以看到：
- 搜索域膨胀的实际效果——逐次附加越来越短的搜索后缀进行 A 记录查询
- 上游转发失败——`connection refused` 和 `i/o timeout`

### 第四步：检查资源使用情况

```bash
kubectl top pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                       CPU(cores)   MEMORY(bytes)
coredns-7d6f9b7c8f-ab12c   385m         287Mi
coredns-7d6f9b7c8f-xy34d   412m         312Mi
```

每个接近 400m CPU——对于 DNS Pod 来说相当高。检查资源限制：

```bash
kubectl get pod -n kube-system coredns-7d6f9b7c8f-ab12c -o yaml | grep -A 6 resources
```

```
resources:
  requests:
    cpu: 100m
    memory: 70Mi
```

只设置了 requests，没有 limits。这意味着 CoreDNS Pod 在节点 CPU 压力下会被内核（CFS）**限流**，在内存压力下可能被**驱逐**。

### 第五步：检查故障 Pod 内的 /etc/resolv.conf

```bash
kubectl exec -n production payment-v1-7b8f9d4c5f-abc12 -- cat /etc/resolv.conf
```

```
nameserver 10.96.0.10
search payment.svc.cluster.local svc.cluster.local cluster.local prod-ns-1.svc.corp.local prod-ns-2.svc.corp.local
options ndots:5
```

可以看到两个问题：
1. **ndots:5**——每次查询导致最多 6 次搜索域查找
2. **search 行过长**——继承了节点自身的 `/etc/resolv.conf`（节点包含公司网络的定制搜索域），这意味着更多的搜索域迭代

### 第六步：对比测试 FQDN 与短名称

在同一 Pod 内：

```bash
# 短名称——6+ 次查询，缓慢，偶尔失败
nslookup payment-svc
# Server: 10.96.0.10
# ** server can't find payment-svc: NXDOMAIN

# FQDN（以点结尾）——单次查询，成功
nslookup payment-svc.production.svc.cluster.local.
# Server: 10.96.0.10
# Address 1: 10.96.0.1
# Name: payment-svc.production.svc.cluster.local
# Address 1: 10.244.3.12 payment-svc.production.svc.cluster.local
```

这证实了搜索域膨胀是瓶颈所在。

### 第七步：检查 CoreDNS ConfigMap

```bash
kubectl get configmap -n kube-system coredns -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus
        forward . /etc/resolv.conf
        cache 30
    }
```

值得注意的问题：
- forward 插件未设置 `max_concurrent`——上游并发连接无限制
- 缓存仅 30 秒——对于 500+ 服务的集群来说 TTL 太短
- 没有自动扩缩容或 Pod 级别的资源限制

---

## 根因

三个相互关联的因素共同导致 DNS 瘫痪：

### 因素 1：CoreDNS Pod 未设置资源限制

CoreDNS Pod 只有 resource requests 而没有 limits。这种配置在生产环境中非常危险：

- **无 CPU 限制**：在节点 CPU 压力下（50 节点集群在高峰期非常常见），CoreDNS Pod 遭受严重的 CFS 节流。本应 <1ms 完成的 DNS 查询可能需要 100ms 以上，甚至直接超时。
- **无内存限制**：节点内存不足时，CoreDNS Pod 成为被驱逐的首要目标。其中一个副本重启 3 次就是因为 OOMKill。

```bash
# 查看 Pod 上次终止原因
kubectl get pod -n kube-system coredns-7d6f9b7c8f-xy34d -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# OOMKilled
```

### 因素 2：默认 ndots:5 导致 6 倍查询放大

在 `ndots:5` 和默认搜索域（外加从节点 resolv.conf 继承的额外域）的作用下，每次短名称 DNS 查找需要发起 **6 次或更多**上游查询才能成功或失败。对于一个每秒处理数千次内部请求的集群：

```
正常情况： 每次查找 1 次查询  →  1000 次查找/秒  →  1000 qps 到 CoreDNS
ndots:5： 每次查找 6 次查询  →  1000 次查找/秒  →  6000 qps 到 CoreDNS
```

这 6 倍放大压垮了已经被节流的 CoreDNS Pod。`forward` 插件无法跟上，导致上游解析器连接超时。

### 因素 3：CoreDNS 副本数严重不足

50 节点集群、500+ 服务只有 2 个 CoreDNS 副本，远远不够。集群没有配置 ClusterProportionalAutoscaler。高峰期每个副本处理约 3000 qps——远超每个 CoreDNS 实例推荐处理的 1000-2000 qps。

---

## 解决方案

### 应急响应（立即缓解）

1. **扩容 CoreDNS 副本**以立即缓解压力：

```bash
kubectl scale deployment -n kube-system coredns --replicas=5
```

2 分钟内，DNS 解析恢复正常。错误率从 15% 降至 <0.1%。

2. **为 CoreDNS Pod 设置资源限制**以防止节流并确保 QoS 等级：

```bash
kubectl edit deployment -n kube-system coredns
```

```yaml
resources:
  requests:
    cpu: 100m
    memory: 70Mi
  limits:
    cpu: 500m
    memory: 500Mi
```

### 修复 ndots 配置

三种方案任选其一即可解决问题：

**方案 A — 全局设置 ndots:1**：

在工作负载的 Pod 模板中配置 `dnsConfig`：

```yaml
spec:
  template:
    spec:
      dnsConfig:
        options:
          - name: ndots
            value: "1"
```

设置 `ndots:1` 后，名称中包含 1 个或以上点号时将直接作为绝对查询处理。像 `payment-svc`（0 个点）仍然会触发搜索域扩展。所以更彻底的方案是 `ndots:0`，或者使用 FQDN。

**方案 B — 应用配置中使用 FQDN（推荐）**：

将应用配置改为使用带尾点的完全限定域名：

```
http://payment-svc.production.svc.cluster.local./api/v1/charge
```

这样可以完全绕过搜索域扩展——一次 DNS 查询，无 ndots 处理。

**方案 C — 修改 Pod DNS 策略**：

```yaml
spec:
  template:
    spec:
      dnsPolicy: "Default"
      dnsConfig:
        options:
          - name: ndots
            value: "1"
```

### 优化 CoreDNS ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 120
    }
```

关键变更：
- **`max_concurrent 1000`**——限制上游并发连接数，防止资源耗尽
- **`cache 120`**——从 30 秒提高到 120 秒，提升缓存命中率

### 添加 ClusterProportionalAutoscaler

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-proportional-autoscaler/master/manifests/coredns.yaml
```

根据集群节点数自动扩缩 CoreDNS 副本：

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: ClusterProportionalAutoscaler
metadata:
  name: coredns-autoscaler
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  options:
    target: "NodeCount"
    base: 2
    max: 8
    nodesPerReplica: 10
```

对于 50 节点：`base(2) + ceil(50/10) = 7` 个副本——足以应对负载。

### 监控与告警

将以下 CoreDNS 指标加入监控体系：

| 指标 | 含义 | 告警阈值 |
|---|---|---|
| `prometheus_coredns_dns_responses_total` | 按 rcode 统计的 DNS 响应总数 | NXDOMAIN 占比 > 5% |
| `prometheus_coredns_dns_request_duration_seconds` | 查询延迟分布 | p99 > 1s |
| `prometheus_coredns_forward_requests_total` | 上游转发请求数 | 丢弃率 > 1% |
| `prometheus_coredns_cache_hits_total` | 缓存效率 | 命中率 < 50% |
| `prometheus_coredns_panics_total` | 插件 panic 次数 | > 0 |

---

## 经验教训

### 做错了什么

1. **CoreDNS 没有资源限制**：一个控制面组件没有限制地运行，在高负载下必然出问题。CoreDNS 尤其敏感，因为 DNS 是同步且面向连接的（UDP 数据流和 TCP 回退）。

2. **默认 ndots:5 太浪费**：Kubernetes 默认的 `ndots:5` 在小集群中工作良好，但在大环境中会造成巨大的查询放大。这个默认值是为了让短名称"开箱即用"，但在规模扩大时代价变得不可接受。

3. **CoreDNS 自动扩缩容不是可选项**：对于超过 10 个节点的集群，手动管理 CoreDNS 副本是不够的。负载随服务数量、请求速率和节点数量动态变化——这些都需要自动扩缩容来应对。

4. **继承节点 resolv.conf 的搜索域**：当节点有自定义 DNS 搜索域时（在企业环境中很常见），这些域会泄漏到 Pod 中，扩大搜索列表，成倍放大 ndots 问题。

### 改进了什么

- CoreDNS 现在同时配置了 **requests 和 limits**，拥有 Guaranteed QoS 等级
- 所有集群工作负载采用 **ndots:1** 或基于 FQDN 的寻址方式
- ClusterProportionalAutoscaler 动态管理 CoreDNS 副本
- 建立了 DNS 监控面板，包含延迟、错误率和缓存命中率
- 在所有团队的 Helm Chart 中标准化配置 Pod 的 `dnsConfig`，显式设置 ndots

---

## 总结

这次事故是三个可管理的问题叠加成生产宕机的完美风暴：

| 问题 | 症状 | 修复 |
|---|---|---|
| CoreDNS 无资源限制 | 节点压力下 Pod 被节流/驱逐 | 设置 CPU/内存 limits，Guaranteed QoS |
| 默认 ndots:5 | 每次短名称查询 6x 放大 | 设置 ndots:1 或使用 FQDN |
| 50 节点仅 2 个 CoreDNS 副本 | 高峰期 qps 压垮 Pod | ClusterProportionalAutoscaler → 7 副本 |

这次故障最具迷惑性的是它的**间歇性特征**——同一服务上一秒解析 `payment-svc` 失败，下一秒可能就成功，完全取决于查询落在哪个 CoreDNS 副本上以及该副本当前是否被节流。这种"海森堡 bug"让 DNS 问题的诊断变得异常困难。

所有修复措施实施后，集群的 DNS p99 延迟从 2.3s 降至 8ms，缓存命中率从约 35% 提升到约 85%。CoreDNS Pod 在 5-7 副本配置下以约 30% CPU 利用率轻松应对峰值流量。

**关键启示**：Kubernetes DNS 不是"免费的基础设施"——它是一个关键路径上的分布式系统，需要合理规划容量、资源保障和显式配置。默认设置（ndots:5，无限制，无自动扩缩容）在开发集群中可以接受，但在生产负载下必然失效。排查间歇性 DNS 故障时，务必检查 Pod 内的 `/etc/resolv.conf`、CoreDNS 资源使用情况和搜索域扩展行为。
