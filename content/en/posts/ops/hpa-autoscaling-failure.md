---
title: "HPA Autoscaling Failure — When Kubernetes Refused to Scale Despite 95% CPU"
date: 2026-06-12
weight: 100530
slug: "hpa-autoscaling-failure"
tags: ["kubernetes", "hpa", "autoscaling", "troubleshooting", "performance"]
categories: ["Troubleshooting"]
description: "A Kubernetes HPA autoscaling failure incident — how a misconfigured metrics pipeline and resource request mismatch caused the HorizontalPodAutoscaler to stop scaling under load, bringing the production service to its knees"
keywords: "kubernetes hpa not scaling, hpa troubleshooting, metrics server not working, k8s autoscaling failed, horizontal pod autoscaler debugging"
draft: false
featured: true
cover:
  image: ""
  caption: "HPA Autoscaling Failure — Troubleshooting"
---

# HPA Autoscaling Failure — When Kubernetes Refused to Scale Despite 95% CPU

## Common Search Queries

If you're landing here from a search, this post covers:

- kubernetes hpa not scaling when cpu is high
- hpa desired replicas stuck at current replicas
- metrics-server not reporting metrics from some nodes
- kubelet certificate rotation failure metrics-server
- hpa targetAverageUtilization calculated across all containers
- sidecar container diluting hpa cpu target
- hpa refuses to scale up partial metrics
- kubectl top pods works but hpa won't scale

---

## The Incident

**Environment**: K8s v1.29, Calico CNI, metrics-server v0.7, 10 worker nodes, 50+ microservices. The affected service was `order-processor` — a critical order processing service with an application container and a Fluentd logging sidecar.

**Time**: Black Friday, 10:00 AM — peak traffic hour.

**Symptoms**: The on-call phone rang within minutes of the traffic spike. The order-processing service latency had exploded from a healthy 200ms to over 30 seconds. Orders were failing with timeout errors. The HPA showed `Current replicas: 3 / Desired: 3` — yet CPU usage on the pods was sitting at 95%.

```bash
# HPA status showed no scaling happening
kubectl get hpa order-processor
NAME              REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
order-processor   Deployment/order-processor    87%/80%   3         10        3          45d
```

The utilization was 87% against an 80% target — barely over the threshold, and only 3 out of a possible 10 replicas were running. The service was drowning.

```bash
# Pod CPU usage told a different story
kubectl top pods -l app=order-processor
NAME                                 CPU(cores)   MEMORY(bytes)
order-processor-7d8f9c6b4c-abc12    950m         256Mi
order-processor-7d8f9c6b4c-def34    948m         244Mi
order-processor-7d8f9c6b4c-ghi56    955m         251Mi
```

Each app container was using ~950m CPU (95% of its 1 CPU request), yet the HPA was complacent. Why?

---

## Background

Before diving into the investigation, let's review how HPA works under the hood.

### HPA Algorithm

The HorizontalPodAutoscaler uses a simple control loop:

```
desiredReplicas = ceil[currentMetricValue / desiredMetricValue] × currentReplicas
```

For CPU utilization, the formula becomes:

```
desiredReplicas = ceil[currentUtilizationPercentage / targetUtilizationPercentage] × currentReplicas
```

In our case: `ceil[87% / 80%] × 3 = ceil[1.0875] × 3 = 2 × 3 = 6` — wait, with 87% utilization against 80% target, that should give `ceil[1.0875] = 2`, meaning `2 × 3 = 6` replicas. So why didn't it scale?

The catch is that the HPA controller recalculates every 15 seconds (default `--horizontal-pod-autoscaler-sync-period`), and with partial metrics from only 7 of 10 nodes reporting, the controller took a conservative approach.

### Metrics Pipeline

The flow of metrics in a Kubernetes cluster follows this path:

```
kubelet (cAdvisor) → metrics-server API → HPA controller
```

1. The kubelet on each node collects resource metrics from cAdvisor
2. metrics-server scrapes each kubelet's `/metrics/resource/v1alpha1` endpoint every 60 seconds
3. The HPA controller queries metrics-server for pod metrics every 15 seconds
4. HPA calculates desired replicas and updates the Deployment's replicas field

If any link in this chain breaks, autoscaling fails silently.

### Resource Metrics vs Custom Metrics

- **Resource metrics**: CPU and memory reported by kubelet/cAdvisor, used with `targetAverageUtilization` or `targetAverageValue`
- **Custom metrics**: Application-level metrics (QPS, latency, queue depth) exposed through adapters like Prometheus Adapter
- **External metrics**: Metrics from outside the cluster (e.g., SQS queue length)

This incident involved only resource metrics.

### Target Types

- **targetAverageUtilization**: Percentage of the pod's resource requests. Calculated as `(current usage / total resource requests) × 100%`
- **targetAverageValue**: Absolute value (e.g., 500m CPU). Does not depend on resource requests.

### Pod Metrics Aggregation

This is the critical part that tripped us up: **HPA calculates utilization against the SUM of ALL containers' resource requests in a pod**.

When you set `targetAverageUtilization: 80`, Kubernetes computes:

```
podCPUUsage = sum of CPU usage across all containers
podCPURequest = sum of CPU requests across all containers
podUtilization = podCPUUsage / podCPURequest × 100%
```

If your pod has an app container requesting 1 CPU and a sidecar requesting 0.1 CPU, the total request is 1.1 CPU. The utilization is calculated against 1.1, not 1.0.

---

## Investigation

What followed was a 45-minute investigation that revealed not one but two compounding issues.

### Step 1: Check HPA Status

```bash
kubectl get hpa order-processor -o wide
```

Output showed `87%/80%` under TARGETS — above threshold but barely. The `REPLICAS` column showed `3`. Something was holding the autoscaler back.

### Step 2: Describe the HPA

```bash
kubectl describe hpa order-processor
```

This revealed important details:

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

`recommended size: 6` — the HPA wanted 6 replicas but was stuck at 3! The `AbleToScale` condition was `True` and `ScalingLimited` was `False`. This combination usually means scaling should happen, but it wasn't.

### Step 3: Check HPA Events

```bash
kubectl get events --field-selector involvedObject.name=order-processor
```

No recent scaling events. The HPA was not emitting scale-up events despite the metric being above threshold. This was suspicious.

### Step 4: Check Current Metrics

```bash
kubectl top pods -l app=order-processor
```

All 3 running pods showed ~950m CPU each. Individual containers:

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

The app was using ~950m, Fluentd was using ~10m. Total pod CPU: ~960m. This looked high enough to trigger a scale-up.

### Step 5: Check If metrics-server Is Working

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

Three nodes (node-03, node-05, node-07) showed 0% across the board — metrics-server could not reach these nodes. This was a critical finding.

### Step 6: Check metrics-server Logs

```bash
kubectl logs -n kube-system deployment/metrics-server
```

The logs showed repeated errors:

```
E1025 10:02:15.123456   1 scraper.go:149] "Failed to scrape node" node="node-03" err="Get \"https://node-03:10250/metrics/resource/v1alpha1\": tls: failed to verify certificate: x509: certificate has expired or is not yet valid"
E1025 10:02:15.234567   1 scraper.go:149] "Failed to scrape node" node="node-05" err="Get \"https://node-05:10250/metrics/resource/v1alpha1\": tls: failed to verify certificate: x509: certificate has expired or is not yet valid"
E1025 10:02:15.345678   1 scraper.go:149] "Failed to scrape node" node="node-07" err="Get \"https://node-07:10250/metrics/resource/v1alpha1\": tls: failed to verify certificate: x509: certificate has expired or is not yet valid"
```

Expired kubelet serving certificates on three nodes were preventing metrics-server from scraping metrics.

### Step 7: Check Kubelet Certificates

On one of the affected nodes:

```bash
journalctl -u kubelet | grep -i certificate | tail -10
```

Output confirmed:

```
Oct 25 09:58:12 node-03 kubelet[1234]: E1025 09:58:12.123456   1234 certificate_manager.go:562] Failed to rotate certificate: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp: lookup kube-api-server on 10.96.0.10:53: read udp 10.244.1.2:43567->10.96.0.10:53: i/o timeout"
```

The kubelet on these nodes had lost connectivity to the API server briefly during a network blip the day before, and the automatic certificate rotation had failed. The serving certificates expired overnight, and by morning — Black Friday — the certificates were invalid.

### Step 8: Check Pod Resource Requests

```bash
kubectl get pod order-processor-7d8f9c6b4c-abc12 -o yaml | grep -A10 resources:
```

Output revealed:

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

The app container requested 1 CPU, and the Fluentd sidecar requested 0.1 CPU. Combined: 1.1 CPU.

### Step 9: Check Sidecar Resource Requests

```bash
kubectl get pod order-processor-7d8f9c6b4c-abc12 \
  -o jsonpath='{.spec.containers[*].resources.requests.cpu}'
```

Output: `1 100m`

### Step 10: Manually Calculate HPA Target

Now let's put it all together.

**App container**: requests 1 CPU, using 0.95 CPU (95%)
**Fluentd sidecar**: requests 0.1 CPU, using 0.01 CPU (10%)

**Total pod request**: 1.0 + 0.1 = 1.1 CPU
**Total pod usage**: 0.95 + 0.01 = 0.96 CPU

**Effective utilization**: 0.96 / 1.1 = 87.3%

**HPA target**: 80%

With `desiredReplicas = ceil[87.3% / 80%] × 3 = ceil[1.091] × 3 = 2 × 3 = 6`.

So the algorithm **should** give 6 replicas. And indeed, `kubectl describe hpa` showed `recommended size: 6`. But the HPA was not scaling. Why?

The answer: **partial metrics**. The HPA controller saw metrics from only 7 out of 10 nodes. With 3 nodes' kubelets not reporting, the metrics-server had incomplete data. One of the 3 existing pods may have been scheduled on an affected node (node-03, node-05, or node-07), meaning its metrics were missing from the calculation.

When the HPA controller cannot obtain metrics for some pods, it falls back to conservative behavior — it does not scale up because it cannot reliably determine whether more replicas are needed. The missing pod's utilization is treated as 0, which artificially lowers the average, preventing the scale-up from triggering.

---

## Root Cause

The incident had two compounding root causes:

### Root Cause 1: Sidecar Container Dilution

The `order-processor` pod had two containers:
- Application container: requests 1 CPU, high utilization
- Fluentd sidecar: requests 0.1 CPU, idle

HPA's `targetAverageUtilization: 80` is calculated against the **total resource requests of ALL containers** in the pod. The idle sidecar's 0.1 CPU request effectively increased the denominator by 10%, diluting the utilization metric.

The app container was at 95% of its own request, but the pod appeared to be at only 87% of combined requests. While the math still suggested scaling to 6 replicas, the dilution made the metric less sensitive — and combined with root cause 2, it prevented scaling entirely.

### Root Cause 2: Expired Kubelet Certificates + Partial Metrics

Three nodes (node-03, node-05, node-07) had expired kubelet serving certificates. The kubelet's automatic certificate renewal had failed due to a transient API server connectivity issue the previous day.

metrics-server v0.7 running on K8s 1.29 required the `--kubelet-use-node-status-port=false` flag to use the node's InternalIP for scraping. Without this flag, metrics-server defaulted to using the hostname, which resolved incorrectly for some nodes, contributing to the scraping failures.

The result: metrics-server could not scrape 3 out of 10 nodes. The HPA controller received partial metrics — pods on those nodes were invisible to the autoscaler. When metrics are missing, the HPA controller errs on the side of caution and refuses to scale up.

### Summary of Contributing Factors

| Factor | Impact |
|--------|--------|
| Sidecar requesting 0.1 CPU | Diluted effective utilization from 95% to 87% |
| Expired kubelet certs on 3 nodes | metrics-server couldn't scrape 30% of nodes |
| Missing `--kubelet-use-node-status-port=false` | metrics-server couldn't resolve node addresses |
| HPA conservative behavior with partial metrics | Refused to scale despite high utilization |
| No HPA failure monitoring | No alert fired when scaling stopped working |

---

## Resolution

We resolved the incident in two phases: emergency and permanent.

### Emergency Fix

**Step 1: Manually scale the deployment**

```bash
kubectl scale deployment order-processor --replicas=10
```

This immediately brought 10 replicas online. Latency dropped back to normal within 2 minutes. Orders resumed processing.

**Step 2: Fix kubelet certificates on affected nodes**

On each affected node (node-03, node-05, node-07):

```bash
# Renew kubelet serving certificates
kubeadm certs renew kubelet-serving

# Restart kubelet to pick up new certificates
systemctl restart kubelet

# Verify certificate validity
journalctl -u kubelet | grep "Certificate rotation" | tail -5
```

**Step 3: Restart metrics-server**

```bash
kubectl rollout restart -n kube-system deployment/metrics-server
```

**Step 4: Verify metrics recovery**

```bash
# All nodes should now show metrics
kubectl top nodes
NAME      CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-01   4528m        56%    28671Mi         72%
node-02   3891m        48%    25123Mi         63%
node-03   2345m        29%    18234Mi         46%
node-04   5123m        64%    30122Mi         76%
# ... all nodes reporting

# HPA should reflect correct metrics
kubectl get hpa order-processor
NAME              REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
order-processor   Deployment/order-processor    85%/80%   3         10        10         45d
```

**Step 5: Add missing metrics-server flag**

Edit the metrics-server deployment:

```bash
kubectl edit deployment -n kube-system metrics-server
```

Add `--kubelet-use-node-status-port=false` to the container args if using K8s v1.29+ with metrics-server v0.7.

### Permanent Fix — HPA Configuration

The real fix was to configure the HPA correctly to target the application container directly, ignoring the sidecar.

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
    # Target app container directly, ignoring sidecar
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

Key changes:

1. **ContainerResource metric**: Targets only the `order-processor` container, ignoring the Fluentd sidecar
2. **Scale-up behavior**: Zero stabilization window for faster response to traffic spikes
3. **Aggressive scale-up policy**: Up to 4 pods per minute, or 100% of current replicas

### Sidecar Resource Request Optimization

For the Fluentd sidecar, we reduced resource requests to the minimum required:

```yaml
resources:
  requests:
    cpu: 25m       # Reduced from 100m
    memory: 64Mi
  limits:
    cpu: 100m
    memory: 128Mi
```

### Long-term Preventive Measures

1. **metrics-server monitoring**: Added alert on `metrics_server_scraper_duration_seconds` and scrape failure count

2. **Kubelet certificate expiry monitoring**: Alert at 30 days before certificate expiry:

```bash
# Check certificate expiry across all nodes
for node in $(kubectl get nodes -o name); do
  echo "=== $node ==="
  kubectl node-shell $node -- openssl x509 -in /var/lib/kubelet/pki/kubelet-server-current.pem -noout -enddate 2>/dev/null
done
```

3. **HPA failure alert**: Prometheus alert rule for HPA failures:

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
      summary: "HPA {{ $labels.horizontalpodautoscaler }} is not scaling"
      description: "HPA {{ $labels.horizontalpodautoscaler }} in {{ $labels.namespace }} has desired {{ $value }} replicas but has not achieved the target for > 5 minutes"

  - alert: MetricsServerDown
    expr: up{job="metrics-server"} == 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "metrics-server is down"
```

4. **Regular HPA testing**: Schedule load tests to validate autoscaling behavior:

```bash
# Generate load to test HPA
kubectl run --rm -i --tty load-generator --image=busybox -- sh -c "
  while true; do
    wget -q -O- http://order-processor:8080/process 2>/dev/null
  done
"
```

5. **`--kubelet-use-node-status-port` configuration**: Ensure this flag is correctly set according to the K8s version. In K8s 1.29+ with metrics-server v0.7, set it to match your cluster's node addressing scheme.

---

## Lessons Learned

### 1. HPA Targets Are Calculated Across All Containers

The most important lesson: **HPA's `targetAverageUtilization` considers ALL containers in a pod**. A sidecar with non-zero resource requests dilutes the utilization metric. If you have sidecars, either:

- Use `autoscaling/v2` with `ContainerResource` metrics targeting only the main container
- Set sidecar resource requests to the minimum possible value
- Use `targetAverageValue` instead of `targetAverageUtilization` to avoid the request-based calculation

### 2. The Metrics Pipeline Is a Single Point of Failure

The entire autoscaling system depends on metrics-server working correctly. If kubelet certificates expire, if the metrics-server pod crashes, or if network issues prevent scraping, the HPA operates blind.

### 3. Monitor the Autoscaler Itself

You need alerts on:
- HPA desired replicas not matching current replicas
- metrics-server scrape failures
- Kubelet certificate expiry
- HPA events (or lack thereof)

### 4. Test Your Autoscaling Under Load

Don't assume the HPA works just because it's configured. Run load tests that push CPU above the target and verify that:
- New pods are created
- Existing pods show correct utilization
- The cluster has headroom for additional replicas
- Scale-down behaves correctly when load drops

### 5. Default HPA Behavior May Be Too Conservative

The default HPA sync period is 15 seconds, but stabilization windows and the conservative approach to missing metrics can significantly delay scaling. For traffic-sensitive services, consider:

- Setting `behavior.scaleUp.stabilizationWindowSeconds: 0`
- Using `ContainerResource` metrics
- Testing with realistic traffic patterns

---

## Summary

### Timeline

| Time | Event |
|------|-------|
| D-1 14:00 | Transient network blip causes kubelet on 3 nodes to lose API server connectivity |
| D-1 14:01 | Kubelet certificate rotation fails silently on 3 nodes |
| D-1 22:00 | Serving certificates expire on 3 nodes |
| D-0 10:00 | Black Friday traffic spike hits order-processing service |
| D-0 10:02 | HPA shows 87% utilization but refuses to scale — partial metrics from only 7/10 nodes |
| D-0 10:03 | Service latency spikes from 200ms to 30s |
| D-0 10:05 | Orders begin failing with timeout errors |
| D-0 10:12 | On-call engineer begins investigation |
| D-0 10:35 | Root cause identified: expired kubelet certs + sidecar dilution |
| D-0 10:37 | Manual scale-up: `kubectl scale deployment order-processor --replicas=10` |
| D-0 10:39 | Latency returns to normal |
| D-0 10:45 | Kubelet certs renewed on all 3 affected nodes |
| D-0 10:47 | metrics-server restarted |
| D-0 11:00 | HPA updated to use `ContainerResource` metric targeting app container only |

### Config Comparison: Before vs After

| Aspect | Before | After |
|--------|--------|-------|
| HPA API Version | autoscaling/v1 (or v2 with Resource metric) | autoscaling/v2 with ContainerResource metric |
| Metric Target | All containers (total 1.1 CPU) | App container only (1 CPU) |
| Scale-up Behavior | Default (5 min stabilization window) | 0 stabilization, aggressive policy |
| Sidecar CPU Request | 100m | 25m |
| metrics-server Flags | Not configured | `--kubelet-use-node-status-port=false` |
| Kubelet Cert Monitoring | None | Prometheus alert at 30 days to expiry |
| HPA Failure Alert | None | Prometheus alert on desired != current for > 5min |
| Load Testing | Manual, ad-hoc | Scheduled monthly load tests with HPA validation |

### Key Commands Reference

```bash
# Check HPA status
kubectl get hpa <name> -o wide

# Detailed HPA info
kubectl describe hpa <name>

# Check HPA events
kubectl get events --field-selector involvedObject.name=<hpa-name>

# Pod metrics with container breakdown
kubectl top pods -l app=<label> --containers

# Node metrics
kubectl top nodes

# Check metrics-server logs
kubectl logs -n kube-system deployment/metrics-server

# Check kubelet certificate status (on node)
journalctl -u kubelet | grep -i certificate

# Manually scale deployment
kubectl scale deployment <name> --replicas=<n>

# Load test HPA
kubectl run --rm -i --tty load-generator --image=busybox -- sh -c "while true; do wget -q -O- http://<service>:<port>/<endpoint> 2>/dev/null; done"
```

### Final Thoughts

The HPA is a powerful tool, but it's not a set-and-forget mechanism. It sits at the intersection of resource management, monitoring infrastructure, and application architecture — and it fails silently when any of these layers has a problem.

In our case, two seemingly minor issues — a sidecar requesting 0.1 CPU and three expired kubelet certificates — combined to paralyze autoscaling during the most critical traffic period of the year. The HPA didn't throw errors. It just quietly stopped doing its job.

Monitor your autoscalers. Test them under realistic load. And remember: in Kubernetes, the thing that monitors everything also needs to be monitored itself.
