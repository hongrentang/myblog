---
title: "K8s HPA 弹性伸缩失效排查——CPU 95% 但拒绝扩容的背后真相"
date: 2026-06-12
weight: 100530
slug: "hpa-autoscaling-failure"
tags: ["kubernetes", "hpa", "autoscaling", "troubleshooting", "performance"]
categories: ["Troubleshooting"]
description: "Kubernetes HPA 弹性伸缩失效排障纪实——一个错误的 metrics 管道配置和资源请求错配，导致 HorizontalPodAutoscaler 在高负载下停止扩容，生产服务濒临崩溃"
keywords: "kubernetes hpa 不扩容, hpa 排查, metrics-server 故障, k8s 弹性伸缩失效, horizontal pod autoscaler 调试, hpa targetAverageUtilization 侧边容器, kubelet 证书过期 metrics-server"
draft: false
featured: true
cover:
  image: ""
  caption: "HPA 弹性伸缩失效排查"
---

# K8s HPA 弹性伸缩失效排查——CPU 95% 但拒绝扩容的背后真相

## 常见搜索词

如果你是通过搜索引擎找到这里的，这篇文章涵盖以下问题：

- kubernetes hpa 不扩容 CPU 高但副本数不变
- hpa desired replicas 卡在当前值不动
- metrics-server 部分节点采集不到指标
- kubelet 证书过期导致 metrics-server 采集失败
- hpa targetAverageUtilization 包含所有容器总和
- sidecar 容器稀释 hpa CPU 目标值
- hpa 部分指标缺失时拒绝扩容
- kubectl top pods 正常但 hpa 不扩缩容

---

## 故障经过

**环境**: K8s v1.29, Calico CNI, metrics-server v0.7, 10 个 Worker 节点, 50+ 微服务。受影响的服务是 `order-processor`——一个关键订单处理服务，包含一个应用容器和一个 Fluentd 日志采集 Sidecar。

**时间**: 黑色星期五，上午 10:00——峰值流量时段。

**症状**: 监控告警在流量高峰后几分钟内响起。order-processing 服务的延迟从健康的 200ms 暴涨到 30 秒以上。订单开始因超时而失败。HPA 显示 `当前副本: 3 / 期望: 3`——但 Pod 的 CPU 使用率高达 95%。

```bash
# HPA 状态显示没有发生扩缩容
kubectl get hpa order-processor
NAME              REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
order-processor   Deployment/order-processor    87%/80%   3         10        3          45d
```

利用率 87%，目标 80%——仅略微超过阈值，而 10 个最大副本只跑了 3 个。服务正在被流量淹没。

```bash
# Pod CPU 使用率却是另一回事
kubectl top pods -l app=order-processor
NAME                                 CPU(cores)   MEMORY(bytes)
order-processor-7d8f9c6b4c-abc12    950m         256Mi
order-processor-7d8f9c6b4c-def34    948m         244Mi
order-processor-7d8f9c6b4c-ghi56    955m         251Mi
```

每个应用容器都在使用约 950m CPU（占其 1 CPU 请求的 95%），但 HPA 无动于衷。为什么？

---

## 背景

在深入排查之前，先回顾一下 HPA 的工作原理。

### HPA 算法

HorizontalPodAutoscaler 使用一个简单的控制循环：

```
期望副本数 = ceil[当前指标值 / 期望指标值] × 当前副本数
```

对于 CPU 利用率，公式变为：

```
期望副本数 = ceil[当前利用率百分比 / 目标利用率百分比] × 当前副本数
```

在我们的场景中：`ceil[87% / 80%] × 3 = ceil[1.0875] × 3 = 2 × 3 = 6`——87% 利用率对比 80% 目标，应该得到 `ceil[1.0875] = 2`，即 `2 × 3 = 6` 个副本。那为什么没有扩容呢？

关键在于 HPA 控制器每 15 秒重新计算一次（默认 `--horizontal-pod-autoscaler-sync-period`），而且由于只有 7/10 个节点上报指标，控制器采取了保守策略。

### 指标采集管道

Kubernetes 集群中的指标流向如下：

```
kubelet (cAdvisor) → metrics-server API → HPA 控制器
```

1. 每个节点上的 kubelet 从 cAdvisor 采集资源指标
2. metrics-server 每 60 秒轮询每个 kubelet 的 `/metrics/resource/v1alpha1` 端点
3. HPA 控制器每 15 秒查询 metrics-server 获取 Pod 指标
4. HPA 计算期望副本数并更新 Deployment 的 replicas 字段

这条链路上的任何一环出现问题，自动伸缩都会静默失效。

### 资源指标 vs 自定义指标

- **资源指标**: kubelet/cAdvisor 报告的 CPU 和内存，用于 `targetAverageUtilization` 或 `targetAverageValue`
- **自定义指标**: 应用级指标（QPS、延迟、队列深度），通过 Prometheus Adapter 等适配器暴露
- **外部指标**: 集群外部的指标（如 SQS 队列长度）

本次事故只涉及资源指标。

### 目标类型

- **targetAverageUtilization**: 占 Pod 资源请求的百分比。计算方式为 `(当前使用量 / 总资源请求量) × 100%`
- **targetAverageValue**: 绝对值（如 500m CPU）。不依赖资源请求

### Pod 指标聚合

这是最关键的认知陷阱：**HPA 计算利用率时，以 Pod 中所有容器的资源请求总和作为分母**。

当你设置 `targetAverageUtilization: 80` 时，Kubernetes 的计算方式是：

```
podCPU使用量 = 所有容器 CPU 使用量之和
podCPU请求量 = 所有容器 CPU 请求量之和
pod利用率 = podCPU使用量 / podCPU请求量 × 100%
```

如果你的 Pod 有一个请求 1 CPU 的应用容器和一个请求 0.1 CPU 的 Sidecar，总请求量为 1.1 CPU。利用率是基于 1.1 计算的，而不是 1.0。

---

## 排查过程

接下来是一段 45 分钟的排查，揭示了两个相互叠加的问题。

### Step 1: 查看 HPA 状态

```bash
kubectl get hpa order-processor -o wide
```

输出显示 TARGETS 列为 `87%/80%`——略高于阈值。REPLICAS 列显示为 `3`。有什么东西在阻止自动伸缩。

### Step 2: 查看 HPA 详情

```bash
kubectl describe hpa order-processor
```

这里揭示了重要细节：

```
Name:                                  order-processor
Namespace:                             default
Reference:                             Deployment/order-processor
Metrics:                               ( current / target )
  resource cpu on pods  (as a percentage of request):  87% (957m) / 80%
Min replicas:                          3
Max replicas:                          10
Deployment pods:                       3 current / 3 desired
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    ReadyForNewScale          recommended size: 6
  ScalingActive  True    ValidMetricFound          the HPA was able to successfully calculate a replica count from cpu resource utilization
  ScalingLimited False   DesiredWithinRange        the desired count is within the acceptable range
```

`recommended size: 6`——HPA 想要 6 个副本，但卡在了 3 个！`AbleToScale` 为 `True`，`ScalingLimited` 为 `False`。这个组合通常意味着应该发生扩容，但实际却没有。

### Step 3: 查看 HPA 事件

```bash
kubectl get events --field-selector involvedObject.name=order-processor
```

没有最近的扩容事件。尽管指标超过了阈值，HPA 却没有发出扩容事件。这很可疑。

### Step 4: 查看当前指标

```bash
kubectl top pods -l app=order-processor
```

3 个运行中的 Pod 各显示约 950m CPU。按容器查看：

```bash
kubectl top pods -l app=order-processor --containers
NAME                              CONTAINER         CPU(cores)   MEMORY(bytes)
order-processor-7d8f9c6b4c-abc12  order-processor   945m         248Mi
order-processor-7d8f9c6b4c-abc12  fluentd           10m          32Mi
order-processor-7d8f9c6b4c-def34  order-processor   948m         246Mi
order-processor-7d8f9c6b4c-def34  fluentd           12m          30Mi
order-processor-7d8f9c6b4c-ghi56  order-processor   950m         250Mi
order-processor-7d8f9c6b4c-ghi56  fluentd           8m           31Mi
```

应用容器约 950m，Fluentd 约 10m。Pod CPU 总计约 960m。这个值看起来足以触发扩容。

### Step 5: 检查 metrics-server 是否正常工作

```bash
kubectl top nodes
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-01   4528m        56%    28671Mi         72%
node-02   3891m        48%    25123Mi         63%
node-03   0m           0%     0Mi             0%
node-04   5123m        64%    30122Mi         76%
node-05   0m           0%     0Mi             0%
node-06   4789m        59%    29234Mi         74%
node-07   0m           0%     0Mi             0%
node-08   4125m        51%    27345Mi         69%
node-09   4987m        62%    31122Mi         78%
node-10   4678m        58%    28123Mi         71%
```

三个节点（node-03、node-05、node-07）全面显示 0%——metrics-server 无法联系到这些节点。这是一个关键发现。

### Step 6: 查看 metrics-server 日志

```bash
kubectl logs -n kube-system deployment/metrics-server
```

日志中反复出现错误：

```
E1025 10:02:15.123456   1 scraper.go:149] "Failed to scrape node" node="node-03" err="Get \"https://node-03:10250/metrics/resource/v1alpha1\": tls: failed to verify certificate: x509: certificate has expired or is not yet valid"
E1025 10:02:15.234567   1 scraper.go:149] "Failed to scrape node" node="node-05" err="Get \"https://node-05:10250/metrics/resource/v1alpha1\": tls: failed to verify certificate: x509: certificate has expired or is not yet valid"
E1025 10:02:15.345678   1 scraper.go:149] "Failed to scrape node" node="node-07" err="Get \"https://node-07:10250/metrics/resource/v1alpha1\": tls: failed to verify certificate: x509: certificate has expired or is not yet valid"
```

三个节点上的 kubelet 服务证书过期，导致 metrics-server 无法采集指标。

### Step 7: 检查 kubelet 证书

在其中一个受影响的节点上：

```bash
journalctl -u kubelet | grep -i certificate | tail -10
```

输出确认：

```
Oct 25 09:58:12 node-03 kubelet[1234]: E1025 09:58:12.123456   1234 certificate_manager.go:562] Failed to rotate certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup kube-api-server on 10.96.0.10:53: read udp 10.244.1.2:43567->10.96.0.10:53: i/o timeout"
```

前一天网络抖动期间，这些节点上的 kubelet 短暂失去了与 API 服务器的连接，自动证书轮换失败。服务证书在夜间过期，到了早上——黑色星期五——证书已经失效。

### Step 8: 检查 Pod 资源请求

```bash
kubectl get pod order-processor-7d8f9c6b4c-abc12 -o yaml | grep -A10 resources:
```

输出显示：

```yaml
    resources:
      requests:
        cpu: "1"
        memory: 512Mi
      limits:
        cpu: "2"
        memory: 1Gi
  - name: fluentd
    image: fluent/fluentd:v1.16
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

应用容器请求 1 CPU，Fluentd Sidecar 请求 0.1 CPU。合计：1.1 CPU。

### Step 9: 检查 Sidecar 资源请求

```bash
kubectl get pod order-processor-7d8f9c6b4c-abc12 \
  -o jsonpath='{.spec.containers[*].resources.requests.cpu}'
```

输出：`1 100m`

### Step 10: 手动计算 HPA 目标

现在来把所有信息拼在一起。

**应用容器**: 请求 1 CPU，使用 0.95 CPU（95%）
**Fluentd Sidecar**: 请求 0.1 CPU，使用 0.01 CPU（10%）

**Pod 总请求**: 1.0 + 0.1 = 1.1 CPU
**Pod 总使用**: 0.95 + 0.01 = 0.96 CPU

**有效利用率**: 0.96 / 1.1 = 87.3%

**HPA 目标**: 80%

根据公式 `期望副本数 = ceil[87.3% / 80%] × 3 = ceil[1.091] × 3 = 2 × 3 = 6`。

所以算法**应该**给出 6 个副本。确实，`kubectl describe hpa` 显示 `recommended size: 6`。但 HPA 为什么没有扩容呢？

答案是：**部分指标缺失**。HPA 控制器只收到了 10 个节点中 7 个的指标。这三个节点的 kubelet 未上报，metrics-server 数据不完整。三个现有 Pod 中可能有一个运行在受影响的节点上（node-03、node-05 或 node-07），这意味着它的指标在计算中被遗漏了。

当 HPA 控制器无法获取某些 Pod 的指标时，它会回退到保守行为——不会进行扩容，因为无法可靠地判断是否需要更多副本。缺失指标的 Pod 被视为利用率为 0，这人为拉低了平均值，阻止了扩容触发。

---

## 根因

本次事故有两个叠加的根因。

### 根因 1: Sidecar 容器稀释

`order-processor` Pod 包含两个容器：
- 应用容器：请求 1 CPU，高利用率
- Fluentd Sidecar：请求 0.1 CPU，空闲状态

HPA 的 `targetAverageUtilization: 80` 是基于 Pod 中**所有容器的总资源请求**计算的。空闲 Sidecar 的 0.1 CPU 请求实际上将分母增加了 10%，稀释了利用率指标。

应用容器达到了其自身请求的 95%，但 Pod 在合并请求中只显示为 87%。虽然数学计算仍然建议扩容到 6 个副本，但稀释效应降低了指标的敏感性——再加上根因 2，最终完全阻止了扩容。

### 根因 2: kubelet 证书过期 + 部分指标

三个节点（node-03、node-05、node-07）的 kubelet 服务证书已过期。前一天 API 服务器短暂的连接故障导致 kubelet 的自动证书续期失败。

在 K8s 1.29 上运行的 metrics-server v0.7 需要 `--kubelet-use-node-status-port=false` 标志才能使用节点的 InternalIP 进行采集。没有这个标志，metrics-server 默认使用主机名，而主机名在某些节点上解析错误，进一步加剧了采集故障。

结果：metrics-server 无法采集 10 个节点中的 3 个。HPA 控制器收到部分指标——这些节点上的 Pod 对自动伸缩器不可见。当指标缺失时，HPA 控制器会谨慎行事，拒绝扩容。

### 根因总结

| 因素 | 影响 |
|------|------|
| Sidecar 请求 0.1 CPU | 有效利用率从 95% 稀释到 87% |
| 3 个节点 kubelet 证书过期 | metrics-server 无法采集 30% 的节点 |
| 缺少 `--kubelet-use-node-status-port=false` | metrics-server 无法解析节点地址 |
| HPA 对部分指标的保守行为 | 高利用率下拒绝扩容 |
| 缺少 HPA 故障监控 | 伸缩失效时无告警触发 |

---

## 解决方案

我们分两个阶段解决事故：紧急修复和长期修复。

### 紧急修复

**第一步：手动扩容 Deployment**

```bash
kubectl scale deployment order-processor --replicas=10
```

立即将 10 个副本上线。延迟在 2 分钟内恢复正常。订单恢复处理。

**第二步：修复受影响节点的 kubelet 证书**

在每个受影响的节点上（node-03、node-05、node-07）：

```bash
# 续期 kubelet 服务证书
kubeadm certs renew kubelet-serving

# 重启 kubelet 加载新证书
systemctl restart kubelet

# 验证证书有效性
journalctl -u kubelet | grep "Certificate rotation" | tail -5
```

**第三步：重启 metrics-server**

```bash
kubectl rollout restart -n kube-system deployment/metrics-server
```

**第四步：验证指标恢复**

```bash
# 所有节点应正常显示指标
kubectl top nodes
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-01   4528m        56%    28671Mi         72%
node-02   3891m        48%    25123Mi         63%
node-03   2345m        29%    18234Mi         46%
node-04   5123m        64%    30122Mi         76%
# ... 所有节点正常上报

# HPA 应反映正确指标
kubectl get hpa order-processor
NAME              REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
order-processor   Deployment/order-processor    85%/80%   3         10        10         45d
```

**第五步：添加缺失的 metrics-server 标志**

编辑 metrics-server Deployment：

```bash
kubectl edit deployment -n kube-system metrics-server
```

在 K8s v1.29+ 配合 metrics-server v0.7 时，在容器参数中添加 `--kubelet-use-node-status-port=false`。

### 永久修复——HPA 配置

真正的修复是正确配置 HPA，直接针对应用容器而非全局 Pod 指标。

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  minReplicas: 3
  maxReplicas: 10
  metrics:
    # 直接定位应用容器，忽略 sidecar
    - type: ContainerResource
      containerResource:
        name: cpu
        container: order-processor
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
      - type: Percent
        value: 100
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 120
```

关键变更：

1. **ContainerResource 指标**：仅针对 `order-processor` 容器，忽略 Fluentd Sidecar
2. **扩容行为**：零稳定窗口，对流量高峰更快响应
3. **积极的扩容策略**：每分钟最多 4 个 Pod，或当前副本数的 100%

### Sidecar 资源请求优化

对于 Fluentd Sidecar，我们将资源请求降到最低：

```yaml
resources:
  requests:
    cpu: 25m       # 从 100m 降低
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

### 长期预防措施

1. **metrics-server 监控**：添加 `metrics_server_scraper_duration_seconds` 和采集失败计数告警

2. **kubelet 证书过期监控**：证书到期前 30 天触发告警：

```bash
# 检查所有节点的证书到期时间
for node in $(kubectl get nodes -o name); do
  echo "=== $node ==="
  kubectl node-shell $node -- openssl x509 -in /var/lib/kubelet/pki/kubelet-server-current.pem -noout -enddate 2>/dev/null
done
```

3. **HPA 故障告警**：Prometheus 告警规则：

```yaml
groups:
- name: hpa-alerts
  rules:
  - alert: HPANotScaling
    expr: |
      (kube_horizontalpodautoscaler_status_desired_replicas{job="kube-state-metrics"}
        != kube_horizontalpodautoscaler_status_current_replicas{job="kube-state-metrics"})
      and on(horizontalpodautoscaler)
      (time() - kube_horizontalpodautoscaler_metadata_generation{job="kube-state-metrics"}) > 300
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "HPA {{ $labels.horizontalpodautoscaler }} 未正常伸缩"
      description: "HPA {{ $labels.horizontalpodautoscaler }} 在 {{ $labels.namespace }} 命名空间中期望 {{ $value }} 副本，但超过 5 分钟未能达到目标"

  - alert: MetricsServerDown
    expr: up{job="metrics-server"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "metrics-server 宕机"
```

4. **定期 HPA 测试**：安排压力测试验证自动伸缩行为：

```bash
# 生成负载测试 HPA
kubectl run --rm -i --tty load-generator --image=busybox -- sh -c "
  while true; do
    wget -q -O- http://order-processor:8080/process 2>/dev/null
  done
"
```

5. **`--kubelet-use-node-status-port` 配置**：确保根据 K8s 版本正确设置此标志。在 K8s 1.29+ 配合 metrics-server v0.7 时，根据集群节点寻址方案设置。

---

## 经验教训

### 1. HPA 目标基于所有容器计算

最重要的教训：**HPA 的 `targetAverageUtilization` 考虑了 Pod 中的全部容器**。带有非零资源请求的 Sidecar 会稀释利用率指标。如果你的 Pod 有 Sidecar，要么：

- 使用 `autoscaling/v2` 的 `ContainerResource` 指标，仅针对主容器
- 将 Sidecar 的资源请求设置为最小可能值
- 使用 `targetAverageValue` 替代 `targetAverageUtilization`，避免基于请求的计算

### 2. 指标采集管道是单点故障

整个自动伸缩系统依赖于 metrics-server 的正常运行。如果 kubelet 证书过期、metrics-server Pod 崩溃或网络问题阻止采集，HPA 就在盲区工作。

### 3. 监控自动伸缩器本身

你需要针对以下情况设置告警：
- HPA 期望副本数不等于当前副本数
- metrics-server 采集失败
- kubelet 证书即将过期
- HPA 事件（或事件缺失）

### 4. 在负载下测试自动伸缩

不要仅仅因为配置了 HPA 就认为它能正常工作。运行压力测试，将 CPU 推过目标阈值，并验证：
- 新 Pod 被创建
- 现有 Pod 显示正确的利用率
- 集群有足够的余量容纳额外副本
- 负载下降时缩容行为正确

### 5. 默认 HPA 行为可能过于保守

默认的 HPA 同步周期是 15 秒，但稳定窗口和缺失指标时的保守策略可能显著延迟扩容。对于流量敏感的服务，考虑：

- 设置 `behavior.scaleUp.stabilizationWindowSeconds: 0`
- 使用 `ContainerResource` 指标
- 使用真实的流量模式进行测试

---

## 总结

### 时间线

| 时间 | 事件 |
|------|------|
| D-1 14:00 | 短暂网络抖动导致 3 个节点 kubelet 断开 API 服务器 |
| D-1 14:01 | 3 个节点上 kubelet 证书轮换静默失败 |
| D-1 22:00 | 3 个节点的服务证书过期 |
| D-0 10:00 | 黑色星期五流量高峰冲击 order-processing 服务 |
| D-0 10:02 | HPA 显示 87% 利用率但拒绝扩容——仅 7/10 节点上报指标 |
| D-0 10:03 | 服务延迟从 200ms 飙升到 30s |
| D-0 10:05 | 订单开始因超时失败 |
| D-0 10:12 | 值班工程师开始排查 |
| D-0 10:35 | 根因定位：kubelet 证书过期 + Sidecar 稀释 |
| D-0 10:37 | 手动扩容：`kubectl scale deployment order-processor --replicas=10` |
| D-0 10:39 | 延迟恢复正常 |
| D-0 10:45 | 3 个受影响节点的 kubelet 证书全部续期 |
| D-0 10:47 | metrics-server 重启 |
| D-0 11:00 | HPA 更新为使用 ContainerResource 指标仅针对应用容器 |

### 配置对比：修复前后

| 项目 | 修复前 | 修复后 |
|------|--------|--------|
| HPA API 版本 | autoscaling/v1（或 v2 使用 Resource 指标） | autoscaling/v2 使用 ContainerResource 指标 |
| 指标目标 | 所有容器（总计 1.1 CPU） | 仅应用容器（1 CPU） |
| 扩容行为 | 默认（5 分钟稳定窗口） | 0 稳定窗口，积极策略 |
| Sidecar CPU 请求 | 100m | 25m |
| metrics-server 标志 | 未配置 | `--kubelet-use-node-status-port=false` |
| kubelet 证书监控 | 无 | Prometheus 到期前 30 天告警 |
| HPA 故障告警 | 无 | Prometheus 期望!=当前超过 5 分钟告警 |
| 压力测试 | 手动、临时进行 | 每月定期测试附带 HPA 验证 |

### 常用命令参考

```bash
# 查看 HPA 状态
kubectl get hpa <名称> -o wide

# 查看 HPA 详情
kubectl describe hpa <名称>

# 查看 HPA 事件
kubectl get events --field-selector involvedObject.name=<hpa名称>

# 按容器查看 Pod 指标
kubectl top pods -l app=<标签> --containers

# 查看节点指标
kubectl top nodes

# 查看 metrics-server 日志
kubectl logs -n kube-system deployment/metrics-server

# 查看 kubelet 证书状态（在节点上执行）
journalctl -u kubelet | grep -i certificate

# 手动扩容 Deployment
kubectl scale deployment <名称> --replicas=<数量>

# 压力测试 HPA
kubectl run --rm -i --tty load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://<服务>:<端口>/<端点> 2>/dev/null; done"
```

### 写在最后

HPA 是一个强大的工具，但它并不是设置好就不用管的机制。它处于资源管理、监控基础设施和应用架构的交汇点——当其中任何一层出现问题时，它都会静默失效。

在我们的案例中，两个看似微小的问题——一个 Sidecar 请求了 0.1 CPU 和三张过期的 kubelet 证书——叠加在一起，在最关键的年度流量高峰期间完全瘫痪了自动伸缩。HPA 没有抛出错误，只是安静地停止了工作。

监控你的自动伸缩器。在真实负载下测试它们。记住：在 Kubernetes 中，监控一切的那个组件本身也需要被监控。
