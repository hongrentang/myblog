---
title: "GPU 显存溢出导致推理服务中断：TorchServe OOM 排查与治理"
date: 2026-05-19
weight: 100040
slug: "gpu-oom-inference-service-troubleshooting"
tags: ["gpu", "ai", "troubleshooting"]
categories: ["AI/ML"]
description: "完整复盘一次 GPU OOM 导致推理服务大面积超时的故障 — 从误判模型加载到定位显存泄漏和批量配置问题"
keywords: "gpu oom, cuda out of memory, 显存溢出排查, torchserve oom, nvidia-smi排查推理服务"
draft: false
featured: true
cover:
  image: "/images/gpu-oom-banner.svg"
  caption: "GPU OOM 排查与治理"
---

# GPU 显存溢出导致推理服务中断：TorchServe OOM 排查与治理

## 问题引导

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- CUDA Out of Memory 排查
- GPU 显存泄漏定位与修复
- TorchServe 推理服务 OOM
- nvidia-smi 解读 GPU 内存使用
- 模型推理批量大小优化
- GPU 内存动态扩容与隔离

---

## 故障现象

**环境**：K8S v1.28.5，NVIDIA GPU Operator v23.9，Tesla T4 × 4 节点，PyTorch 2.1 + TorchServe 0.10，模型为 BERT-base 多分类（~420MB），推理服务通过 Istio 暴露。

**时间**：周二 14:00，业务高峰期。模型推理量约为平时的 2.3 倍（促销活动）。

**症状**：监控告警——AI 推理服务的 P99 延迟从 120ms 飙升至 8.5s，错误率从 0.1% 升至 23%。部分请求返回 500，日志中出现 `CUDA Out of Memory`。TorchServe 进程存活但推理请求持续超时。

```bash
# Prometheus 告警信息
{__name__="nginx_ingress_controller_request_duration_seconds", quantile="0.99"} 8.54
{__name__="ai_service_errors_total"} 1273  (rate: 23.4/s)

# Pod 状态
NAME                                   READY   STATUS             RESTARTS   AGE
bert-inference-b8f9d4c5f-a1b2          0/1     CrashLoopBackOff   3          12m
bert-inference-b8f9d4c5f-c3d4          1/1     Running            0          15m
bert-inference-b8f9d4c5f-e5f6          0/1     CrashLoopBackOff   2          12m
```

**影响**：促销活动期间用户无法使用 AI 功能（智能推荐、自动分类），前端大量报错。业务方反馈用户体验严重受损，转化率下降约 15%。

---

## 误判阶段

### 第一反应：模型加载失败

看到 Pod CrashLoopBackOff，第一反应是新部署的模型版本有问题。检查了模型存储路径和版本号，模型文件完整、MD5 一致。

### 接着怀疑 GPU Operator 异常

怀疑是 NVIDIA GPU Operator 的 Device Plugin 出了问题，导致 GPU 不可用：

```bash
kubectl describe pod bert-inference-b8f9d4c5f-a1b2
```

结果发现 Pod 确实分配了 GPU：

```
Resources:
  limits:
    nvidia.com/gpu: 1
```

但 Events 没有明显错误。真正的问题藏在 Pod 日志里——而我没有第一时间去看。

### 浪费了 5 分钟检查节点 GPU 状态

我登录节点执行 `nvidia-smi`，看到 GPU 占用率 97%、显存使用 14.8GB/15.8GB，得出结论"GPU 很忙"——但没有进一步判断"这是正常推理负载还是异常泄漏"。

```bash
# 误判：看到 GPU 忙就以为是正常高负载
# 实际上需要对比基线：14.8GB 远高于正常推理时的 6-8GB
nvidia-smi
```

**如何避免这个误判**：应该直接对比当前显存与正常基线的差值，而不是只看百分比。正常推理服务显存使用应该是稳定的，如果持续增长就是泄漏。

---

## 排查过程

### Step 1: 检查推理日志 — 发现 CUDA OOM

```bash
kubectl logs bert-inference-b8f9d4c5f-f1g2 -n ai-production --tail 200
```

日志输出：

```
2026-05-19 14:02:15 INFO  Received request: /predictions/bert_classify, batch_size=32
2026-05-19 14:02:15 INFO  Loading model for inference...
2026-05-19 14:02:15 ERROR Prediction failed: CUDA Out Of Memory. Tried to allocate 512.00 MiB (GPU 0; 15.75 GiB; total capacity: 15.75 GiB; already allocated: 14.21 GiB; free: 184.00 MiB; peak: 15.10 GiB)
2026-05-19 14:02:15 ERROR Request failed: <class 'torch.cuda.OutOfMemoryError'>
```

**如何解读**：关键指标是 `already allocated: 14.21 GiB` 和 `peak: 15.10 GiB`。T4 总显存 15.75 GiB，当前已分配 14.21 GiB，尝试额外分配 512MB 失败。显存已经被占用了 90%+，正常推理服务基线应该在 6-8GB 左右。

这个错误说明两件事：
1. 当前显存使用远高于正常水平（14.21 GiB vs 基线 6-8 GiB）
2. 尝试分配新张量时已经没有空间

### Step 2: 检查 nvidia-smi — 确认显存状态

```bash
# 进入 Pod 查看 GPU 状态
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- nvidia-smi
```

输出：

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

**如何解读**：
- `14810MiB / 15360MiB` — 显存占用 96%，仅剩 550MiB
- GPU-Util 97% — GPU 几乎满负荷（正常推理服务在 60-80%）
- Temperature 68°C — 偏高但正常（T4 最大 85°C）
- 这个 Pod 应该只有 ~420MB 的模型 + 约 2GB 的推理工作集，总使用应在 3GB 以内
- 14.8GB 的显存使用意味着存在**严重泄漏**或**多个进程共用 GPU**

### Step 3: 检查进程级 GPU 使用 — 发现僵尸进程

```bash
# 查看 GPU 上正在运行的进程
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- nvidia-smi pmon -c 1
```

或者更详细地：

```bash
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- fuser -v /dev/nvidia*
```

然后查看容器内所有 Python 进程：

```bash
kubectl exec -it bert-inference-b8f9d4c5f-f1g2 -n ai-production -- ps aux | grep python
```

输出：

```
root     12345  2.1  8.2  ...  python torchserve --start
root     18923  0.0  8.2  ...  python <defunct>
root     18924  0.0  8.2  ...  python <defunct>
root     18925  0.0  8.2  ...  python <defunct>
```

**发现**：主 TorchServe 进程（PID 12345）正常，但存在 3 个僵尸（defunct）Python 进程。这些进程在 `torch.cuda.empty_cache()` 之前就被 kill 了，导致它们的 GPU 内存从未被释放。

**如何判断泄漏**：通过 `nvidia-smi pmon` 可以看到每个进程的 GPU 内存使用。僵尸进程的显存不会被回收——CUDA 上下文在进程完全销毁后才释放，而僵尸进程没有完全退出。

### Step 4: 分析推理处理逻辑 — 找到内存泄漏点

检查推理服务的请求处理代码：

```python
# 推理服务 handler 简化代码
class BertClassifierHandler(BaseHandler):
    def initialize(self, context):
        super().initialize(context)
        self.model = self.load_model()  # 加载 BERT 模型
        self.batch_size = context.manifest.get("batch_size", 32)
        
    def preprocess(self, data):
        # 问题：每次请求创建新的张量但未及时释放
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
        # torch.cat 会创建新的张量，累积显存
        return torch.cat(inputs).to(self.device)
    
    def inference(self, data):
        with torch.no_grad():
            # 问题：outputs 变量在每个请求中创建，但前一个请求的 outputs
            # 可能因某些异常没有释放（比如客户端超时断开）
            outputs = self.model(data)
            # 问题：max_seq_length=512 加上 batch_size=32，
            # 峰值显存占用 = 512 * 32 * 768 * 4bytes * 12layers ≈ 6GB
            # 加上模型本身 420MB + 激活缓存 ≈ 3GB
            # 理论上总使用 ~9-10GB
            return outputs
```

**根因分析**：

1. **显存泄漏路径**：TorchServe 的 Worker 在处理请求时，如果客户端在推理完成前断开连接（HTTP 超时），异常处理路径没有执行 `torch.cuda.empty_cache()`。多次超时后，各个 Worker 积累了大量未释放的中间张量。

2. **batch_size 过大**：`batch_size=32` + `max_length=512` 的组合在峰值时产生约 6GB 的中间张量。正常推理场景中，batch_size=8 即可满足吞吐需求，32 是业务方设置的"为了促销活动提高吞吐"，但效果适得其反——批处理越大，峰值显存越高，越容易 OOM。

3. **事件链**：
   - 促销活动流量增长 2.3 倍 → 部分请求处理变慢 → 客户端超时断开 → Worker 异常终止 → 中间张量未释放 → 显存碎片化 → 新请求无法分配显存 → OOM → Pod CrashLoopBackOff

---

## 恢复过程

### 紧急恢复（5 分钟）

第一步：重启推理服务，立即恢复线上能力。

```bash
# 1. 快速扩容（增加 Pod 分散负载）
kubectl scale deployment bert-inference -n ai-production --replicas=6

# 2. 滚动重启所有 Pod（清理显存）
kubectl rollout restart deployment bert-inference -n ai-production

# 3. 验证恢复
kubectl rollout status deployment bert-inference -n ai-production -w
```

验证恢复是否成功：

```bash
# 检查 Pod 全部 Running
kubectl get pods -n ai-production -l app=bert-inference

# 检查推理延迟是否恢复
kubectl exec -it bert-inference-xxx -n ai-production -- bash -c "curl -s -o /dev/null -w '%{time_total}s\n' http://localhost:8080/predictions/bert_classify -d '{\"text\":\"test\"}'"

# 监控显存是否回到基线（应降至 6-8GB 左右）
kubectl exec -it bert-inference-xxx -n ai-production -- nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits
```

检查显存输出应为 `6144`~`8192` MiB，如果仍然 >12000 MiB 说明泄漏持续存在。

### 降低批处理大小（缓解，10 分钟）

将 batch_size 从 32 降低到 8，减少峰值显存占用：

```bash
# 修改 TorchServe 配置
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

同时调整 Deployment 环境变量：

```yaml
env:
- name: BATCH_SIZE
  value: "8"
- name: MAX_SEQ_LENGTH
  value: "256"  # 从 512 降到 256，显存再减半
- name: CUDA_LAUNCH_BLOCKING
  value: "0"
```

重启生效：

```bash
kubectl rollout restart deployment bert-inference -n ai-production
```

验证：推理延迟应回落到 200ms 以内，显存稳定在 5-7GB，不再持续增长。

---

## 长期修复方案

### 1. 修复显存泄漏：添加异常处理的显存清理

在推理 handler 中添加保护逻辑：

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
            # 异常时立即清理显存
            torch.cuda.empty_cache()
            raise e
    
    def inference(self, data):
        try:
            with torch.no_grad():
                outputs = self.model(data)
                return outputs
        except torch.cuda.OutOfMemoryError:
            # 捕获 OOM，清理后重试一次
            torch.cuda.empty_cache()
            gc.collect()
            # 如果还 OOM，降级到更小的 batch
            if data.shape[0] > 1:
                half = data.shape[0] // 2
                return torch.cat([
                    self.inference(data[:half]),
                    self.inference(data[half:])
                ])
            raise
        finally:
            # 每次推理后清理临时变量
            if 'outputs' in locals():
                del outputs
            if torch.cuda.memory_allocated() > 8 * 1024**3:  # >8GB
                torch.cuda.empty_cache()
```

**为什么能防复发**：异常路径和正常路径都显式执行了显存清理，并且 OOM 时自动降级到更小的 batch 处理，不会因为单次 OOM 导致整个 Worker 挂掉。

### 2. Pod 级别 GPU 显存隔离

使用 NVIDIA MPS（Multi-Process Service）或设置 CUDA 可见设备上限：

```yaml
# Deployment 配置
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
            # GPU 显存硬限制（依赖 GPUOperator 的 GMM）
          requests:
            nvidia.com/gpu: 1
        env:
        - name: CUDA_VISIBLE_DEVICES
          value: "0"
        - name: PYTORCH_CUDA_ALLOC_CONF
          value: "max_split_size_mb:128"
```

**为什么能防复发**：`PYTORCH_CUDA_ALLOC_CONF=max_split_size_mb:128` 限制了 CUDA 内存分配器的最大分片大小，避免碎片化导致的显存假满。再加上 GMM（GPU Memory Manager）可设置 Pod 级显存上限，一个 Pod 泄漏不会影响同节点其他 Pod。

### 3. 设置 Prometheus GPU 显存告警

配置 GPU 显存使用率告警，在达到临界值之前提前发现：

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
      summary: "GPU {{ $labels.gpu }} 显存使用率超过 85%"
      description: "当前使用率: {{ $value | humanizePercentage }}"
      runbook: "https://internal.runbooks/gpu-oom"

  - alert: GPUMemoryGrowth
    expr: |
      avg_over_time(nvidia_gpu_memory_used_bytes[15m]) -
      avg_over_time(nvidia_gpu_memory_used_bytes[60m]) > 2*1024*1024*1024
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "GPU 显存持续增长超过 2GiB（可能泄漏）"
```

**为什么能防复发**：第一个规则在显存使用率超过 85% 时发出警告（留出 15% 缓冲），第二个规则通过对比 15 分钟和 60 分钟的平均值来检测显存持续增长趋势——这是泄漏的典型特征。

### 4. 推理服务优雅降级

在推理服务中添加熔断降级机制：

```python
# 推理服务入口处添加限流
import torch
from functools import lru_cache

class RateLimiter:
    def __init__(self, max_concurrent=4):
        self.semaphore = threading.Semaphore(max_concurrent)
        
    def __enter__(self):
        acquired = self.semaphore.acquire(timeout=30)
        if not acquired:
            raise TimeoutError("推理服务繁忙，请稍后重试")
    
    def __exit__(self, *args):
        self.semaphore.release()

# 在请求处理时
def handle_request(self, data):
    with RateLimiter(max_concurrent=4):
        return self._do_inference(data)
```

配合 K8S HPA 自动扩容：

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
        averageValue: 8589934592  # 8GiB
```

**为什么能防复发**：限流防止突发流量打垮服务，HPA 根据显存使用自动扩容。当单个 Pod 显存超过 8GiB 时自动扩容分散负载。

### 5. 推理服务独立 GPU 节点池

将推理服务调度到独立的 GPU 节点池，隔离训练和推理工作负载：

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
# Taint 防止训练 Pod 调度到推理节点
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

**为什么能防复发**：训练任务（特别是分布式训练）通常会占满整个 GPU 显存。如果训练和推理混部，训练任务一启动推理服务就 OOM。独立的 T4 推理节点池确保了推理服务的资源不被训练任务抢占。

---

## 复现步骤

在测试环境复现 GPU OOM 场景：

1. 部署推理服务配置 `batch_size=32`、`max_seq_length=512`
2. 使用压测工具发送并发推理请求：
   ```bash
   # 使用 hey 或 wrk 发送并发请求
   hey -n 10000 -c 50 -m POST \
     -H "Content-Type: application/json" \
     -d '{"text":"test message for inference service"}' \
     http://inference-service:8080/predictions/bert_classify
   ```
3. 监控显存变化：
   ```bash
   watch -n 1 'kubectl exec -it <pod> -n ai-production -- nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits'
   ```
4. 观察显存持续增长直至 OOM

---

## 排查命令速查

```bash
# 查看 GPU 整体状态
nvidia-smi
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu,temperature.gpu --format=csv

# 查看 GPU 进程级显存占用
nvidia-smi pmon -c 1
fuser -v /dev/nvidia*

# Pod 内检查
kubectl exec -it <pod> -n <ns> -- nvidia-smi
kubectl exec -it <pod> -n <ns> -- ps aux | grep python
kubectl exec -it <pod> -n <ns> -- cat /proc/$(pidof python)/status | grep VmRSS

# 查看推理日志
kubectl logs <pod> -n ai-production --tail 200 | grep -i "cuda\|oom\|error"

# PyTorch 显存诊断（在 Pod 内执行 python）
python -c "
import torch
print(f'Allocated: {torch.cuda.memory_allocated()/1024**3:.2f} GiB')
print(f'Cached: {torch.cuda.memory_reserved()/1024**3:.2f} GiB')
print(f'Max allocated: {torch.cuda.max_memory_allocated()/1024**3:.2f} GiB')
"

# 清空显存缓存
python -c "
import torch
import gc
gc.collect()
torch.cuda.empty_cache()
"

# Prometheus 查询 GPU 显存
# nvidia_gpu_memory_used_bytes{gpu="0"}
```

---

## 总结

排查链路：

```
推理延迟飙高 + 500 错误
  → 检查 Pod（部分 CrashLoopBackOff、部分 Running）
  → 查看日志（CUDA Out of Memory）
  → nvidia-smi（显存 96%，远高于基线）
  → 发现僵尸 Python 进程（异常断开未释放显存）
  → 分析代码（异常路径无显存清理 + batch_size 过大）
  → 重启 Pod + 调整 batch_size → 恢复
```

总耗时：从告警到恢复 12 分钟。前 5 分钟浪费在 GPU Operator 排查上。

**核心教训**：

1. **显存泄漏需要主动检测**：不能等 OOM 告警，应该通过 Prometheus 监控显存增长趋势（15min vs 60min 均值对比），持续增长说明泄漏

2. **异常路径必须清理显存**：Python 的 `try/finally` + `torch.cuda.empty_cache()` 是推理服务的标配。客户端断开、请求超时等异常场景必须走清理逻辑

3. **Batch size 不是越大越好**：批处理越大峰值显存越高，收益边际递减。找到最佳性价比点（通常 4-8），而不是盲目设 32 或 64

4. **推理和训练必须隔离**：共享 GPU 节点池是 OOM 的头号元凶。独立的推理节点池 + Taint/Toleration 是必备的

5. **Pod 重启是止血手段，不是治本**：CrashLoopBackOff 的自动重启可以恢复服务，但如果根本问题（泄漏）没修，只会反复 OOM
