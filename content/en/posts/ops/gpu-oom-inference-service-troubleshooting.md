---
title: "GPU OOM Causing Inference Service Outage: TorchServe Out of Memory Troubleshooting"
date: 2026-05-19
weight: 100040
slug: "gpu-oom-inference-service-troubleshooting"
tags: ["gpu", "ai", "troubleshooting"]
categories: ["AI/ML"]
description: "Full postmortem of a GPU OOM causing inference service timeouts — from misdiagnosing model loading to pinpointing memory leaks and batch size misconfiguration"
keywords: "gpu oom, cuda out of memory, gpu memory leak troubleshooting, torchserve oom, nvidia-smi inference troubleshooting"
draft: false
featured: true
cover:
  image: "/images/gpu-oom-banner.svg"
  caption: "GPU OOM Troubleshooting & Mitigation"
---

# GPU OOM Causing Inference Service Outage: TorchServe Out of Memory Troubleshooting

## Common Search Queries

If you're landing here from a search, this post covers:

- CUDA Out of Memory troubleshooting
- GPU memory leak detection and fix
- TorchServe inference OOM
- nvidia-smi GPU memory analysis
- inference batch size optimization
- GPU memory isolation and limits

---

## The Incident

**Environment**: K8S v1.28.5, NVIDIA GPU Operator v23.9, Tesla T4 × 4 nodes, PyTorch 2.1 + TorchServe 0.10, BERT-base multi-classification model (~420MB), inference service exposed via Istio.

**Time**: Tuesday 14:00, peak business hours. Model inference traffic was 2.3× normal volume due to a promotion campaign.

**Symptoms**: Monitoring alerted — AI inference service P99 latency jumped from 120ms to 8.5s, error rate rose from 0.1% to 23%. Partial requests returned 500 with `CUDA Out of Memory` errors in logs. TorchServe processes were still alive but inference requests kept timing out.

```bash
# Prometheus alert data
{__name__="nginx_ingress_controller_request_duration_seconds", quantile="0.99"} 8.54
{__name__="ai_service_errors_total"} 1273  (rate: 23.4/s)

# Pod status
NAME                                   READY   STATUS             RESTARTS   AGE
bert-inference-b8f9d4c5f-a1b2          0/1     CrashLoopBackOff   3          12m
bert-inference-b8f9d4c5f-c3d4          1/1     Running            0          15m
bert-inference-b8f9d4c5f-e5f6          0/1     CrashLoopBackOff   2          12m
```

**Impact**: Users couldn't use AI features (smart recommendations, auto-classification) during the promotion. Frontend was throwing errors. Business reported a ~15% conversion rate drop.

---

## Misdiagnosis

### First Reaction: Model Loading Failed

Seeing CrashLoopBackOff, my first thought was a broken model deployment. I checked the model storage path and version — the model file was intact, MD5 matched.

### Next Suspect: GPU Operator Issue

I suspected the NVIDIA GPU Operator's Device Plugin had failed, making GPUs unavailable:

```bash
kubectl describe pod bert-inference-b8f9d4c5f-f1g2 -n ai-production
```

The pod did have a GPU allocated:

```
Resources:
  limits:
    nvidia.com/gpu: 1
```

But Events showed no obvious errors. The real problem was hidden in the pod logs — which I didn't check first.

### Wasted 5 Minutes on Node-Level GPU Check

I SSH'd into the node and ran `nvidia-smi`, saw 97% GPU utilization and 14.8GiB/15.8GiB memory used, and concluded "GPU is busy" — without checking whether this was normal inference load or a leak.

```bash
# Mistake: assuming high GPU usage = normal load
# Should have compared against baseline: 14.8GiB vs normal 6-8GiB
nvidia-smi
```

**How to avoid this**: Compare current memory usage against the known baseline, not just the percentage. A healthy inference service has stable memory usage — continuous growth is a leak.

---

## Investigation

### Step 1: Check Inference Logs — Found CUDA OOM

```bash
kubectl logs bert-inference-b8f9d4c5f-f1g2 -n ai-production --tail 200
```

Output:

```
2026-05-19 14:02:15 INFO  Received request: /predictions/bert_classify, batch_size=32
2026-05-19 14:02:15 INFO  Loading model for inference...
2026-05-19 14:02:15 ERROR Prediction failed: CUDA Out Of Memory. Tried to allocate 512.00 MiB (GPU 0; 15.75 GiB; total capacity: 15.75 GiB; already allocated: 14.21 GiB; free: 184.00 MiB; peak: 15.10 GiB)
2026-05-19 14:02:15 ERROR Request failed: <class 'torch.cuda.OutOfMemoryError'>
```

**How to read this**: `already allocated: 14.21 GiB` and `peak: 15.10 GiB`. T4 has 15.75 GiB total — 90%+ already consumed, only 184 MiB free. The normal baseline for this service is 6-8 GiB. This tells us two things:
1. Current memory is far above baseline (14.21 GiB vs 6-8 GiB)
2. There's no room for even a 512 MiB tensor allocation

### Step 2: Check nvidia-smi — Confirm Memory Exhaustion

```bash
# Check GPU state from inside the pod
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- nvidia-smi
```

Output:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.105.17   Driver Version: 525.105.17   CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla T4            Off  | 00000000:00:09.0 Off |                    0 |
| N/A   68C    P0    N/A / 70W |  14810MiB / 15360MiB |     97%      Default |
+-------------------------------+----------------------+----------------------+
```

**How to read this**:
- `14810MiB / 15360MiB` — 96% memory used, only 550MiB free
- GPU-Util 97% — GPU is near saturation (normal inference: 60-80%)
- Temperature 68°C — warm but acceptable (T4 max is 85°C)
- This service should use ~420MB for the model + ~2GB working set = ~3GB total
- 14.8GB usage signals **severe leakage** or **multiple processes sharing the GPU**

### Step 3: Check Process-Level GPU Usage — Found Zombie Processes

```bash
# Check processes using GPU
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- nvidia-smi pmon -c 1
```

More detailed:

```bash
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- fuser -v /dev/nvidia*
```

Then check all Python processes:

```bash
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- ps aux | grep python
```

Output:

```
root     12345  2.1  8.2  ...  python torchserve --start
root     18923  0.0  8.2  ...  python <defunct>
root     18924  0.0  8.2  ...  python <defunct>
root     18925  0.0  8.2  ...  python <defunct>
```

**Discovery**: The main TorchServe process (PID 12345) was alive, but 3 zombie (defunct) Python processes existed. These were TorchServe worker processes that were killed during `torch.cuda.empty_cache()` — their GPU memory was never released because they never fully exited.

**How to identify a leak**: `nvidia-smi pmon` shows per-process GPU memory. Zombie processes hold their CUDA context until they're fully reaped. A zombie Python process will keep its GPU memory permanently allocated.

### Step 4: Analyze the Inference Handler — Find the Leak

The inference handler code:

```python
class BertClassifierHandler(BaseHandler):
    def initialize(self, context):
        super().initialize(context)
        self.model = self.load_model()
        self.batch_size = context.manifest.get("batch_size", 32)
        
    def preprocess(self, data):
        # Problem: creates new tensors on every request without cleanup
        inputs = []
        for row in data:
            tokens = self.tokenizer(
                row["text"], 
                padding=True, 
                truncation=True,
                max_length=512,
                return_tensors="pt"
            )
            inputs.append(tokens)
        return torch.cat(inputs).to(self.device)
    
    def inference(self, data):
        with torch.no_grad():
            # Problem: outputs from previous requests may not be freed
            # if client disconnects before completion
            outputs = self.model(data)
            return outputs
```

**Root cause chain**:

1. **Memory leak path**: When TorchServe workers handle requests and the client disconnects before inference completes (HTTP timeout), the exception handler doesn't call `torch.cuda.empty_cache()`. Over time, accumulated intermediate tensors from timed-out requests remain allocated.

2. **Oversized batch**: `batch_size=32` + `max_length=512` peaks at ~6GB of intermediate tensors per inference. Normal batch_size=8 would use ~1.5GB. The business team increased it "for higher promotion throughput" — but larger batches mean higher peak memory, increasing OOM risk.

3. **Event chain**:
   - Promotion traffic 2.3× normal → request processing slows → client timeouts → workers terminate abnormally → intermediate tensors not freed → GPU memory fragmentation → new allocations fail → OOM → Pod CrashLoopBackOff

---

## Recovery

### Emergency Restore (5 minutes)

Scale up and restart immediately to restore service:

```bash
# 1. Scale up to spread the load
kubectl scale deployment bert-inference -n ai-production --replicas=6

# 2. Rolling restart to clear GPU memory
kubectl rollout restart deployment bert-inference -n ai-production

# 3. Verify recovery
kubectl rollout status deployment bert-inference -n ai-production -w
```

Verify the fix:

```bash
# Check all pods are Running
kubectl get pods -n ai-production -l app=bert-inference

# Test inference latency
kubectl exec -it bert-inference-xxx -n ai-production -- bash -c "curl -s -o /dev/null -w '%{time_total}s\n' http://localhost:8080/predictions/bert_classify -d '{\"text\":\"test\"}'"

# Check memory returned to baseline (should be ~6144-8192 MiB)
kubectl exec -it bert-inference-xxx -n ai-production -- nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits
```

Expected: memory used is 6144-8192 MiB. If still >12000 MiB, the leak persists.

### Reduce Batch Size (Mitigation, 10 minutes)

Lower batch size from 32 to 8 to reduce peak memory:

```bash
kubectl edit configmap bert-inference-config -n ai-production
```

```yaml
# config.properties
inference_address=http://0.0.0.0:8080
management_address=http://0.0.0.0:8081
metrics_address=http://0.0.0.0:8082
number_of_netty_threads=4
job_queue_size=100
model_store=/models
model_snapshot={"name":"bert_classify","version":"1.0","batchSize":8,"maxBatchDelay":100}
```

Also reduce sequence length in the Deployment:

```yaml
env:
- name: BATCH_SIZE
  value: "8"
- name: MAX_SEQ_LENGTH
  value: "256"
- name: CUDA_LAUNCH_BLOCKING
  value: "0"
```

Restart:

```bash
kubectl rollout restart deployment bert-inference -n ai-production
```

Verify: latency should drop to under 200ms, GPU memory stable at 5-7GB with no growth trend.

---

## Long-Term Fixes

### 1. Fix the Memory Leak: Add Exception-Handled Memory Cleanup

Add cleanup logic to the inference handler:

```python
class BertClassifierHandler(BaseHandler):
    def initialize(self, context):
        super().initialize(context)
        self.model = self.load_model()
        self.device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        self.model.to(self.device)
        self.model.eval()
        
    def preprocess(self, data):
        try:
            inputs = []
            for row in data:
                tokens = self.tokenizer(
                    row["text"],
                    padding=True,
                    truncation=True,
                    max_length=int(os.getenv("MAX_SEQ_LENGTH", "256")),
                    return_tensors="pt"
                )
                inputs.append(tokens)
            return torch.cat(inputs).to(self.device)
        except Exception as e:
            torch.cuda.empty_cache()
            raise e
    
    def inference(self, data):
        try:
            with torch.no_grad():
                outputs = self.model(data)
                return outputs
        except torch.cuda.OutOfMemoryError:
            torch.cuda.empty_cache()
            gc.collect()
            if data.shape[0] > 1:
                half = data.shape[0] // 2
                return torch.cat([
                    self.inference(data[:half]),
                    self.inference(data[half:])
                ])
            raise
        finally:
            if 'outputs' in locals():
                del outputs
            if torch.cuda.memory_allocated() > 8 * 1024**3:
                torch.cuda.empty_cache()
```

**Why this prevents recurrence**: Every code path (success, exception, OOM) explicitly cleans up GPU memory. On OOM, it automatically degrades to half-batch processing instead of crashing.

### 2. Pod-Level GPU Memory Isolation

Use NVIDIA MPS or CUDA allocation constraints:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bert-inference
  namespace: ai-production
spec:
  template:
    spec:
      containers:
      - name: inference
        image: inference-service:2.1
        resources:
          limits:
            nvidia.com/gpu: 1
        env:
        - name: CUDA_VISIBLE_DEVICES
          value: "0"
        - name: PYTORCH_CUDA_ALLOC_CONF
          value: "max_split_size_mb:128"
```

**Why this prevents recurrence**: `PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:128` limits CUDA allocator fragment size, preventing "false full" from fragmentation. Combined with GPUOperator's GMM, a single leaking pod won't affect neighbors on the same node.

### 3. Prometheus GPU Memory Alerts

Alert before hitting the OOM threshold:

```yaml
groups:
- name: gpu-alerts
  rules:
  - alert: GPUHighMemoryUsage
    expr: |
      (nvidia_gpu_memory_used_bytes / nvidia_gpu_memory_total_bytes) * 100 > 85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "GPU {{ $labels.gpu }} memory usage above 85%"
      description: "Current usage: {{ $value | humanizePercentage }}"
      runbook: "https://internal.runbooks/gpu-oom"

  - alert: GPUMemoryGrowth
    expr: |
      avg_over_time(nvidia_gpu_memory_used_bytes[15m]) -
      avg_over_time(nvidia_gpu_memory_used_bytes[60m]) > 2*1024*1024*1024
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "GPU memory growing >2GiB (possible leak)"
```

**Why this prevents recurrence**: Rule 1 warns at 85% (15% headroom buffer). Rule 2 detects growth trends by comparing 15-min vs 60-min averages — the hallmark of a memory leak.

### 4. Graceful Degradation with Rate Limiting

Add rate limiting and circuit breaking:

```python
class RateLimiter:
    def __init__(self, max_concurrent=4):
        self.semaphore = threading.Semaphore(max_concurrent)
        
    def __enter__(self):
        acquired = self.semaphore.acquire(timeout=30)
        if not acquired:
            raise TimeoutError("Inference service busy, retry later")
    
    def __exit__(self, *args):
        self.semaphore.release()

def handle_request(self, data):
    with RateLimiter(max_concurrent=4):
        return self._do_inference(data)
```

Configure autoscaling based on GPU memory:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bert-inference-hpa
  namespace: ai-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bert-inference
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: nvidia_gpu_memory_used_bytes
      target:
        type: AverageValue
        averageValue: 8589934592
```

**Why this prevents recurrence**: Rate limiting prevents traffic spikes from overwhelming the service. HPA scales out when GPU memory exceeds 8GiB per pod, distributing load before OOM occurs.

### 5. Dedicated Inference GPU Node Pool

Isolate inference from training workloads:

```yaml
apiVersion: v1
kind: Node
metadata:
  labels:
    gpu-type: inference
    gpu-pool: t4-inference
---
apiVersion: v1
kind: Pod
spec:
  nodeSelector:
    gpu-type: inference
    gpu-pool: t4-inference
---
apiVersion: v1
kind: Node
spec:
  taints:
  - key: gpu-pool
    value: inference
    effect: NoSchedule
  tolerations:
  - key: gpu-pool
    value: inference
    operator: Equal
    effect: NoSchedule
```

**Why this prevents recurrence**: Training jobs typically consume 100% of GPU memory. Co-locating training and inference on the same node guarantees OOM when a training job starts. A dedicated T4 inference pool ensures inference resources are never preempted.

---

## Reproduction Steps

To reproduce GPU OOM in a test environment:

1. Deploy inference service with `batch_size=32`, `max_seq_length=512`
2. Send concurrent inference requests:
   ```bash
   hey -n 10000 -c 50 -m POST \
     -H "Content-Type: application/json" \
     -d '{"text":"test message for inference service"}' \
     http://inference-service:8080/predictions/bert_classify
   ```
3. Monitor memory growth:
   ```bash
   watch -n 1 'kubectl exec -it <pod> -n ai-production -- nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits'
   ```
4. Observe memory climbing until OOM

---

## Quick Reference

```bash
# GPU overall status
nvidia-smi
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv

# Per-process GPU memory
nvidia-smi pmon -c 1
fuser -v /dev/nvidia*

# Inside the pod
kubectl exec -it <pod> -n <ns> -- nvidia-smi
kubectl exec -it <pod> -n <ns> -- ps aux | grep python
kubectl exec -it <pod> -n <ns> -- cat /proc/$(pidof python)/status | grep VmRSS

# Check inference logs
kubectl logs <pod> -n ai-production --tail 200 | grep -i "cuda\|oom\|error"

# PyTorch memory diagnostics
kubectl exec -it <pod> -n ai-production -- python -c "
import torch
print(f'Allocated: {torch.cuda.memory_allocated()/1024**3:.2f} GiB')
print(f'Cached: {torch.cuda.memory_reserved()/1024**3:.2f} GiB')
print(f'Max allocated: {torch.cuda.max_memory_allocated()/1024**3:.2f} GiB')
"

# Clear CUDA cache
kubectl exec -it <pod> -n ai-production -- python -c "
import torch, gc; gc.collect(); torch.cuda.empty_cache()
"

# Prometheus query
# nvidia_gpu_memory_used_bytes{gpu="0"}
```

---

## Summary

Investigation chain:

```
Latency spike + 500 errors
  → Checked pods (partial CrashLoopBackOff)
  → Found CUDA Out of Memory in logs
  → nvidia-smi (96% memory, far above baseline)
  → Found zombie Python processes (aborted workers not freeing memory)
  → Analyzed handler code (no cleanup on exception path + oversized batch_size)
  → Restarted pods + reduced batch_size → Recovered
```

Total time from alert to recovery: 12 minutes. First 5 minutes wasted investigating GPU Operator.

**Key Lessons**:

1. **Detect leaks proactively, don't wait for OOM**: Monitor GPU memory growth trends (15min vs 60min average), not just absolute usage. Continuous growth = leak.

2. **Always clean up GPU memory in exception paths**: `try/finally` + `torch.cuda.empty_cache()` is mandatory for inference services. Every timeout, disconnect, or error must trigger cleanup.

3. **Bigger batch size is NOT better**: Peak memory scales with batch size, with diminishing throughput returns. Find the sweet spot (typically 4-8), don't blindly set 32 or 64.

4. **Isolate inference from training**: Shared GPU node pools are the #1 cause of inference OOM. Dedicated inference node pools + Taints/Tolerations are essential.

5. **Restarting is a止血 (stop the bleeding), not a cure**: CrashLoopBackOff auto-restart restores service temporarily, but if the underlying leak isn't fixed, OOM will recur.
