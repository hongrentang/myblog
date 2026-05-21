---
title: "Pod OOMKilled Repeatedly — From Raising Limits to Finding JVM Off-Heap Memory"
date: 2026-05-21
weight: 100120
slug: "pod-oomkilled-jvm-container-memory"
tags: ["kubernetes", "java", "memory", "troubleshooting"]
categories: ["K8S"]
description: "Full postmortem of Pod OOMKilled caused by JVM off-heap memory — from mindlessly raising limits to finding thread stacks and Direct Buffer"
keywords: "pod oomkilled, jvm container memory, kubernetes java oom, k8s memory limits, off-heap memory"
draft: false
featured: true
cover:
  image: "/images/pod-oomkilled-banner.svg"
  caption: "Pod OOMKilled Troubleshooting"
---

# Pod OOMKilled Repeatedly — From Raising Limits to Finding JVM Off-Heap Memory

## Symptoms

Java service kept restarting after deployment.

```bash
kubectl get pods -n production
```

```
NAME                          READY   STATUS      RESTARTS   AGE
order-service-v1-abc12        0/1     OOMKilled   12         45m
```

12 restarts. Pod detail:

```bash
kubectl describe pod order-service-v1-abc12 -n production
```

```
Containers:
  order-service:
    Limits:
      memory: 512Mi
    Last State:   Terminated
      Reason:     OOMKilled
      Exit Code:  137
```

`OOMKilled` + `Exit Code 137` (SIGKILL) — container exceeded its memory limit and the cgroup OOM killer terminated it.

**Impact**: Order service intermittently unavailable. Each restart causes 10-15s of downtime.

## Investigation

### Wrong Turn 1: Blindly Increased memory limit

OOMKilled? Just add more memory.

```bash
kubectl set resources deployment order-service-v1 \
  -c order-service \
  --limits=memory=1Gi
```

Pod restarted. Ran for 2 hours, then OOMKilled again.

```bash
kubectl set resources deployment order-service-v1 \
  -c order-service \
  --limits=memory=2Gi
```

Lasted 4 hours. OOMKilled again.

**Lesson**: Mindlessly raising limits doesn't fix the problem — memory keeps growing. And bigger limits mean fewer pods per node. A 2Gi pod when you only need 512Mi wastes 75% of cluster capacity.

### Wrong Turn 2: Only Checked JVM Heap

"Let me check JVM heap usage."

```bash
kubectl exec order-service-v1-abc12 -n production -- jcmd 1 GC.heap_info
```

```
jcmd: command not found
```

No JDK tools in the container. Try `jstat`:

```bash
kubectl exec order-service-v1-abc12 -n production -- jstat -gc 1
```

```
 OC        OU        MC     MU
174784.0  123456.0  45678.0 41234.0
```

Old generation = ~120MB. Young gen ≈ 70MB. total heap ≈ 200MB. Far from the limit. So why OOMKilled?

**Lesson**: JVM memory = heap + off-heap (Metaspace, thread stacks, Direct Buffer, JNI, Code Cache, etc.). Heap was only 200MB, but off-heap was eating 800MB+.

### Wrong Turn 3: Tuned GC Parameters

"Maybe GC fragmentation?"

```yaml
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

Still OOMKilled. GC was fine — heap was stable at ~200MB.

```bash
kubectl exec order-service-v1-abc12 -n production -- cat /proc/1/status | grep VmRSS
```

```
VmRSS:  1876543 kB
```

**RSS 1.8GB**. Container limit was only 512MiB (at the time). JVM was using 1.8GB with heap at only 200MB — off-heap was the killer.

## Root Cause

```bash
kubectl exec order-service-v1-abc12 -n production -- jcmd 1 VM.native_memory summary
```

```
Native Memory Tracking:
Total: reserved=2456MB, committed=1876MB
- Java Heap (reserved=512MB, committed=200MB)
- Thread (reserved=512MB, committed=512MB)
- Other (reserved=512MB, committed=512MB)    # Direct Buffer etc.
- Unknown (reserved=256MB, committed=256MB)
```

| Area | Usage | Why |
|------|-------|-----|
| Heap | 200MB | Actual app usage |
| Thread | 512MB | 500+ threads × 1MB default stack |
| Other | 512MB | Direct Buffer (Netty, gRPC, NIO) |

The worst part — **JVM wasn't aware of container limits**.

```bash
kubectl exec order-service-v1-abc12 -n production -- java -XX:+PrintFlagsFinal 2>/dev/null | grep MaxHeapSize
```

```
size_t MaxHeapSize = 4294967296 {product} {ergonomic}
```

`MaxHeapSize = 4GB`. JVM thought it had the host's memory (16GB host → JVM default = 1/4 = 4GB), but the container limit was 512MiB.

**Combined issues**:
1. `UseContainerSupport` not active — JVM saw host memory, not cgroup limits
2. Default 1MB thread stack × 500+ threads = 512MB
3. Direct Buffer + Netty + gRPC allocating off-heap
4. Container limit was only 512MiB — off-heap consumed everything before heap could

## Solutions

### Option A: Container-Aware JVM (Recommended)

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: order-service
        resources:
          requests:
            memory: 2Gi
          limits:
            memory: 2Gi
        env:
        - name: JAVA_OPTS
          value: >
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
            -Xss256k
            -XX:MaxDirectMemorySize=256m
            -XX:MaxMetaspaceSize=128m
```

| Flag | Effect |
|------|--------|
| `UseContainerSupport` | JVM respects cgroup limits (JDK 8u191+, default on since JDK 10) |
| `MaxRAMPercentage=75` | JVM uses max 75% of container limit — leaves 25% for off-heap |
| `Xss256k` | Thread stack from 1MB → 256KB. Huge savings with many threads |
| `MaxDirectMemorySize=256m` | Caps Direct Buffer (defaults to Xmx, which is too large) |

With 2Gi limit: heap ≈ 1.5Gi (75%) + 500×256KB threads ≈ 125MB + Direct Buffer 256MB = fits perfectly.

### Option B: Verify JVM Config

```bash
# Check JDK version
kubectl exec order-service-v1-abc12 -n production -- java -version

# Verify container-aware heap sizing
kubectl exec order-service-v1-abc12 -n production -- java -XX:+PrintFlagsFinal 2>/dev/null | grep -E "MaxHeapSize|UseContainerSupport"
```

```
bool UseContainerSupport = true
size_t MaxHeapSize = 1073741824  # 1Gi
```

### Verify Recovery

```bash
# 1. Pod stays Running
kubectl get pods -n production -w
# Expected: Running, RESTARTS not incrementing

# 2. Memory usage within limit
kubectl top pod order-service-v1-abc12 -n production
# Expected: MEMORY stable below 1.6Gi

# 3. Native memory check
kubectl exec order-service-v1-abc12 -n production -- jcmd 1 VM.native_memory summary
# Expected: Total committed < container limit 2Gi
```

✅ **Recovery verification**:
- Pod status `Running`, RESTARTS stable
- `kubectl top pod` shows memory at 70-80% of limit
- Business API returns 200 OK

### Option C: Long-Term Prevention

```dockerfile
FROM openjdk:11-jre-slim
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -Xss256k"
COPY target/app.jar app.jar
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

```bash
# Monitor: alert when RSS > 85% of limit
# Prometheus: container_memory_working_set_bytes / kube_pod_container_resource_limits > 0.85
```

## What I Learned

1. **OOMKilled doesn't mean Java heap is full.** Off-heap memory (thread stacks, Direct Buffer, Native Memory) is the silent killer. `cat /proc/1/status | grep VmRSS` is the fastest way to check actual memory usage.

2. **Blindly raising limits wastes cluster capacity.** Every GiB you add reduces node density. Profile the app's real memory, then set limits accordingly.

3. **JVM in containers needs explicit memory configuration.** JDK 8u191+ and JDK 11+ default to `UseContainerSupport`, but if your base image is older JDK 8, JVM uses host memory. Verify with `PrintFlagsFinal`.

4. **Thread stacks are the most overlooked off-heak consumer.** 500 threads × 1MB = 512MB. At 256KB stack, that's 128MB — saving 384MB with one flag change.

5. **K8S OOMKilled debug checklist**: Pod Events → `kubectl top pod` → `/proc/1/status | grep VmRSS` → `jcmd VM.native_memory`. Don't start by raising limits.
