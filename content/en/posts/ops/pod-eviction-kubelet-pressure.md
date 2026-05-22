---
title: "Pod Eviction Deep Dive — When Kubelet Decides Your Pod Doesn't Belong"
date: 2026-05-22
weight: 100140
slug: "pod-eviction-kubelet-pressure"
tags: ["kubernetes", "k8s", "pod-eviction", "node-pressure", "troubleshooting"]
categories: ["K8S"]
description: "Pods repeatedly evicted — from checking logs and node resources to discovering disk pressure and missing container log rotation"
keywords: "pod evicted, node pressure, kubelet eviction, disk pressure, container log rotation, kubernetes troubleshooting"
draft: false
featured: true
cover:
  image: "/images/pod-eviction-banner.svg"
  caption: "Pod Eviction Troubleshooting"
---

# Pod Eviction Deep Dive — When Kubelet Decides Your Pod Doesn't Belong

## The Incident

3 PM. Alert fires: "Deployment payment-worker available replicas below threshold."

Check the cluster:

```bash
kubectl get pods -n payment
```

```
NAME                              READY   STATUS   RESTARTS   AGE
payment-worker-7d9f8c6b8f-abc12   0/1     Evicted  0          12m
payment-worker-7d9f8c6b8f-def34   0/1     Evicted  0          8m
payment-worker-7d9f8c6b8f-ghi56   0/1     Evicted  0          3m
payment-worker-7d9f8c6b8f-jkl78   1/1     Running  0          2m
payment-worker-7d9f8c6b8f-mno90   0/1     Evicted  0          15m
```

A trail of Evicted pods. payment-worker pods across three worker nodes have been getting evicted one by one. Only the one that just started is still Running.

> Environment: K8S 1.28, 3 masters + 5 workers, each worker 128GB RAM / 20 cores. payment-worker is CPU-intensive, each pod requests 2c4g.

**Impact**: Payment reconciliation latency jumped from 5 minutes to 40 minutes. Massive backlog. No user-facing impact yet (async worker), but operations is already asking "where's today's reconciliation report."

## Investigation

### Wrong turn 1: Check pod logs

First instinct — pod crashed. Let's see what the logs say.

```bash
kubectl logs -n payment payment-worker-7d9f8c6b8f-abc12
```

```
Error from server (BadRequest): previous terminated container "payment-worker" not found
```

The evicted pod's logs are already gone. Try `--previous`:

```bash
kubectl logs -n payment payment-worker-7d9f8c6b8f-abc12 --previous
```

Same result.

**Lesson learned**: Evicted != CrashLoopBackOff. A crashing pod still has logs you can read. An evicted pod has been killed by the kubelet — it's like your landlord changed the locks and threw your stuff out. The logs went with the pod.

### Wrong turn 2: Check node resource usage

Maybe the node is out of resources.

```bash
kubectl top nodes
```

```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
worker-1   12500m       62%    78256Mi         61%
worker-2   9840m        49%    65432Mi         51%
worker-3   14200m       71%    90123Mi         70%
worker-4   6200m        31%    45200Mi         35%
worker-5   5000m        25%    32000Mi         25%
```

Looks fine? Highest is worker-3 at 71% CPU, 70% memory. 90GB used out of 128GB — 38GB free.

Plenty of headroom. Why is kubelet evicting pods?

**Lesson learned**: `kubectl top node` only shows CPU and memory. But kubelet eviction monitors **three** resources: memory pressure, **disk pressure**, and PID pressure. I only checked two. Missed the most important one.

### Wrong turn 3: Rollout restart

Not sure what's wrong — "reboot it."

```bash
kubectl rollout restart deployment payment-worker -n payment
```

All pods restart and get rescheduled to different nodes. Looks fixed.

Two hours later: same alert. New pods are getting evicted again.

**Lesson learned**: Eviction is a node-level behavior. Rebuilding pods doesn't fix the underlying node pressure — it just makes you run in circles. If the root cause (disk pressure, memory pressure) isn't resolved, the new pods will get evicted too.

### The real finding: node conditions and events

Now I'm looking at the right place — the node itself.

```bash
kubectl describe node worker-3
```

The Conditions section:

```
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  MemoryPressure       False   Thu, 22 May 2026 14:30:15 +0800   Thu, 22 May 2026 10:15:00 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         True    Thu, 22 May 2026 14:30:15 +0800   Thu, 22 May 2026 09:45:00 +0800   KubeletHasDiskPressure       kubelet has disk pressure
  PIDPressure          False   Thu, 22 May 2026 14:30:15 +0800   Thu, 22 May 2026 09:45:00 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
```

**`DiskPressure: True`** — there it is.

It's been in DiskPressure since 09:45 AM — almost 5 hours. Kubelet has been trying to reclaim disk space all this time.

Check the node events:

```bash
kubectl describe node worker-3 | grep -A 10 Events
```

```
Events:
  Normal   NodeHasSufficientMemory  5m    kubelet  Node worker-3 status is now: NodeHasSufficientMemory
  Normal   NodeHasDiskPressure      5h    kubelet  Node worker-3 status is now: NodeHasDiskPressure
  Normal   NodeHasSufficientDisk    10m   kubelet  Node worker-3 status is now: NodeHasSufficientDisk
  Normal   NodeHasDiskPressure      8m    kubelet  Node worker-3 status is now: NodeHasDiskPressure
```

Disk pressure is oscillating — the classic eviction thrashing pattern. Kubelet evicts a pod → disk frees up briefly → new pod arrives → disk fills up again → evict again.

SSH to the node:

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        98G   88G   5.4G  95% /
/dev/sdb        500G  200G  300G  40% /data
```

Root partition at 95%!

```bash
# What's eating the space?
du -sh /var/log/pods/
```

```
45G     /var/log/pods/
```

```bash
du -sh /var/lib/containers/
```

```
30G     /var/lib/containers/
```

Container logs: 45GB. Container images and layers: 30GB. Together they consume 75GB.

```bash
# Which pod logs are the worst?
du -sh /var/log/pods/*payment*
```

```
12G     /var/log/pods/payment_worker-abc123
8G      /var/log/pods/payment_worker-def456
...
```

payment-worker logs are the biggest — 12GB for a single pod. Zero log rotation configured.

```bash
# Check kubelet eviction thresholds
cat /var/lib/kubelet/config.yaml | grep -i eviction
```

```
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

Root partition (nodefs) has only 5.4GB free (~5.5%), well below the 10% hard threshold. That triggers DiskPressure and kubelet starts evicting pods.

**Why payment-worker was targeted**: Kubelet ranks pods by QoS class when evicting:

1. BestEffort (no resource limits) → evicted first
2. Burstable (partial limits) → evicted second
3. Guaranteed (full limits) → evicted last

Check payment-worker's QoS:

```bash
kubectl get pod payment-worker-7d9f8c6b8f-abc12 -n payment -o yaml | grep -A 6 resources
```

```yaml
resources:
  requests:
    cpu: "2"
    memory: 4Gi
  limits:
    cpu: "4"
    memory: 8Gi
```

requests != limits → **Burstable** QoS. Not the lowest priority, but most infra components (coredns, calico-node) are Guaranteed. So payment-worker became the second wave of evictions.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct | Root partition at 95% utilization, triggering kubelet DiskPressure eviction threshold |
| Root | No container log rotation configured — logs accumulated to 45GB over days |
| Trigger | Unbounded log growth → root partition over 90% → kubelet evicts pods on nodefs below QoS priority |
| Thrashing | Eviction frees small amount of log space → disk briefly under threshold → new pods arrive → logs grow → evict again |

The full eviction cycle:

```
Container logs fill /var/log/pods/ 
  → nodefs usage > 90% (evictionHard)
  → kubelet sets NodeDiskPressure=True
  → kubelet evicts pods by QoS: BestEffort → Burstable → Guaranteed
  → Evicted pod's logs get cleaned → nodefs briefly recovers
  → New pod scheduled in → logs accumulate again → cycle repeats
```

## Fix

### Emergency: clean up and reclaim space

```bash
# 1. Clean logs from evicted pods
journalctl --vacuum-size=200M

# 2. Remove unused container images
crictl rmi --prune

# 3. Truncate oversized log files (careful — confirm which ones)
find /var/log/pods/ -name "*.log" -size +100M -exec truncate -s 0 {} \;

# 4. Drain the node to force full recovery
kubectl drain worker-3 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon worker-3
```

```bash
# Verify reclaim
df -h /
```

```
/dev/sda1        98G   78G   15G   80% /
```

From 95% to 80%. DiskPressure cleared:

```bash
kubectl describe node worker-3 | grep DiskPressure
```

```
DiskPressure         False   ...
```

### Root fix: container log rotation

This is the one thing that prevents recurrence. Configure Docker or containerd log rotation:

For **containerd** (modern K8S default):

```bash
cat >> /etc/containerd/config.toml << 'EOF'
[plugins."io.containerd.grpc.v1.cri".logging]
  max_size = "50MB"
  max_files = 5
EOF

systemctl restart containerd
```

For **Docker** runtime:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
```

```bash
cat >> /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5"
  }
}
EOF

systemctl restart docker
```

**Why this prevents recurrence**:
- Caps per-container logs at 50MB × 5 files = 250MB max
- Before: one pod could generate 12GB of logs with no cap
- Rotation keeps logs bounded without intervention
- This is "stop the room from heating up" instead of just "spray water when the alarm goes off"

### Hardening: tune kubelet eviction thresholds

The defaults are too permissive for production (waiting until 10% free is too late):

```yaml
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "15%"
  nodefs.inodesFree: "10%"
  imagefs.available: "20%"

evictionSoft:
  nodefs.available: "20%"
evictionSoftGracePeriod:
  nodefs.available: "2m"
```

Add monitoring alerts:

```bash
# Prometheus alert rules
# - node:node_filesystem_usage > 0.8 → warn early (don't wait for 90%)
# - kubelet_node_status_condition{condition="DiskPressure",status="true"} → page immediately
```

### Verification

```bash
# 1. Confirm all nodes have DiskPressure=False
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name} {.status.conditions[?(@.type=="DiskPressure")].status}{"\n"}{end}'
```

```
worker-1 False
worker-2 False
worker-3 False
worker-4 False
worker-5 False
```

```bash
# 2. Confirm no more Evicted pods
kubectl get pods -n payment | grep -c Evicted
# 0

kubectl get pods -n payment
# All Running

# 3. Verify log rotation
ls -lh /var/log/pods/payment_worker-*/payment-worker/*.log
# No files > 50MB

# 4. Reconciliation latency back to normal
# Dashboard: 40min → 5min
```

✅ **Recovery confirmed**:
- All nodes DiskPressure=False
- Zero Evicted pods
- Log files capped at 50MB
- Reconciliation latency back to normal

## What I Learned

1. **Evicted ≠ CrashLoopBackOff.** Two completely different mechanisms. CrashLoopBackOff: the process inside the pod is dying — check pod logs. Evicted: the node kicked your pod out — check node conditions. I wasted 20 minutes running `kubectl logs` on evicted pods before realizing the direction was wrong.

2. **`kubectl top node` lies by omission.** It only shows CPU and memory. Kubelet eviction monitors three dimensions: memory, disk, and PID. Miss one and you miss the root cause. When debugging node issues, **Conditions** in `kubectl describe node` is the first thing to read.

3. **Container log rotation is not optional.** So many cluster bootstrap scripts forget this. Default behavior is **unbounded log growth**. A chatty service running for a week can fill a root partition. Configure rotation before going to production — regardless of your container runtime.

4. **QoS class determines eviction order, not resource usage.** On a node with 50 pods, the one getting evicted isn't necessarily the one using the most memory — it's the one with the lowest QoS class. If your critical service is Burstable or BestEffort, it will be among the first evicted when pressure hits.

5. **"Restart" is not a fix for eviction cycles.** Rebuilding the deployment makes it look better temporarily, but the root cause (disk pressure from unbounded logs) will catch up. Node-pressure problems don't go away until the pressure source is eliminated.
