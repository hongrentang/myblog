---
title: "CUDA Version Mismatch — When Your Container Outruns Your Driver"
date: 2026-05-25
weight: 100160
slug: "cuda-version-mismatch-troubleshooting"
tags: ["cuda", "gpu", "nvidia", "ai", "troubleshooting"]
categories: ["AI/ML"]
description: "Deploying Triton Inference Server to a GPU node — pod starts fine, but CUDA programs crash with 'driver version is insufficient' due to container CUDA version exceeding the host driver's capability"
keywords: "cuda version mismatch, nvidia driver, cuda driver insufficient, gpu inference, triton server"
draft: false
featured: true
cover:
  image: "/images/cuda-version-mismatch-banner.svg"
  caption: "CUDA Version Mismatch Troubleshooting"
---

# CUDA Version Mismatch — When Your Container Outruns Your Driver

## The Incident

Tuesday morning. A new model needs to go live on Triton Inference Server. The K8S cluster has 4 GPU nodes, each with 4× A100 80GB, driver version 525.85.05.

Deployment YAML is ready, `kubectl apply -f`. The pod gets scheduled to a GPU node. `kubectl get pods` shows Running — but Triton never comes up. CrashLoopBackOff.

```bash
kubectl get pods -n inference
```

```
NAME                           READY   STATUS             RESTARTS   AGE
triton-server-7d9f8c6b8f-x1    0/1     CrashLoopBackOff   4          5m
```

Pod starts (not Pending, not ContainerCreating), so GPU resources are allocated. But it crashes within seconds.

```bash
kubectl logs -n inference triton-server-7d9f8c6b8f-x1
```

```
I0525 09:30:15.123456 1 server.cc:632] 
+--------+---------------------------------------------------------+
| Option | Value                                                   |
+--------+---------------------------------------------------------+
| CUDA   | CUDA driver version 12.0  (insufficient)                |
|        | CUDA runtime version 12.2                                |
+--------+---------------------------------------------------------+
E0525 09:30:15.123789 1 cuda_utils.cc:98] CUDA driver version 12.0 is insufficient for CUDA runtime version 12.2
E0525 09:30:15.123890 1 cuda_utils.cc:101] Please upgrade your NVIDIA driver or downgrade your CUDA toolkit
```

The error says it all: **CUDA driver version 12.0 is insufficient for CUDA runtime version 12.2**.

**Impact**: Model can't deploy. Downstream services blocked. The team spent the whole morning suspecting a dead GPU or broken driver.

## Investigation

### Wrong turn 1: GPU must be dead

CUDA error means GPU problem, right? Run nvidia-smi:

```bash
kubectl exec -n inference triton-server-7d9f8c6b8f-x1 -- nvidia-smi
```

```
Tue May 25 09:35:00 2026
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 525.85.05    Driver Version: 525.85.05    CUDA Version: 12.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  NVIDIA A100 80GB     On  | 00000000:00:04.0 Off |                    0 |
|  0%   35C    P0    45W / 300W |      0MiB / 81920MiB |      0%      Default |
|                               |                      |             Disabled |
+-------------------------------+----------------------+----------------------+
```

GPU is visible. Temperature normal. Memory free. Persistence mode on.

**Lesson learned**: `nvidia-smi` showing GPU info does NOT mean CUDA programs will work. nvidia-smi uses the NVML API (management layer), not CUDA runtime. They're different code paths. The GPU driver loaded fine, the card isn't dead, memory is available — but the CUDA runtime in the container is newer than what the driver supports. Think of it like a phone that boots fine (driver works) but can't run an app requiring a newer OS version (CUDA runtime).

### Wrong turn 2: Out of GPU memory

Not a hardware issue. Maybe OOM?

```bash
kubectl logs -n inference triton-server-7d9f8c6b8f-x1 --tail=50 | grep -i "memory"
```

No memory-related errors. Resource limits look fine:

```bash
kubectl describe pod -n inference triton-server-7d9f8c6b8f-x1 | grep -A 10 "Limits"
```

```
    Limits:
      cpu:     8
      memory:  32Gi
      nvidia.com/gpu: 1
    Requests:
      cpu:     4
      memory:  16Gi
      nvidia.com/gpu: 1
```

1× A100 (80GB), model only needs 12GB. Plenty of headroom.

### Wrong turn 3: Rebuild with different image tag

"Try a different tag." Switch from `23.12` to `23.10`:

```yaml
image: nvcr.io/nvidia/tritonserver:23.10-py3
```

```bash
kubectl delete pod -n inference triton-server-xxx
# Deployment recreates
```

Same result — CrashLoopBackOff.

```bash
kubectl logs -n inference triton-server-新Pod
```

Same error: "CUDA driver version 12.0 is insufficient for CUDA runtime version 12.2."

**Lesson learned**: Blindly switching Triton image tags doesn't help when you don't know the CUDA version each tag embeds. Triton 23.12 carries CUDA 12.2. Triton 23.10 also carries CUDA 12.2. You're rolling dice unless you check the mapping.

### The real finding: CUDA version mismatch

Let's actually read the error carefully:

```
| CUDA   | CUDA driver version 12.0  (insufficient)                |
|        | CUDA runtime version 12.2                                |
```

Driver supports CUDA 12.0. Container has CUDA 12.2.

**Host side**:

```bash
# SSH to the GPU node
nvidia-smi | grep "CUDA Version"
```

```
| NVIDIA-SMI 525.85.05    Driver Version: 525.85.05    CUDA Version: 12.0 |
```

```bash
nvidia-smi --query-gpu=driver_version --format=csv,noheader
```

```
525.85.05
```

**Container side**:

```bash
kubectl exec -n inference triton-server-xxx -- nvcc --version
```

```
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2023 NVIDIA Corporation
Built on Wed_Nov_22_10:17:15_PST_2023
Cuda compilation tools, release 12.2, V12.2.0
```

Container has CUDA 12.2. Driver supports 12.0 max.

NVIDIA driver 525 series max supported CUDA version: 12.0. CUDA 12.2 requires driver ≥ 535.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct | Container CUDA runtime 12.2 > host driver's supported CUDA version 12.0 |
| Root | Triton image `23.12-py3` embeds CUDA 12.2; node has NVIDIA driver 525 (max CUDA 12.0) |
| Trigger | Using the latest Triton image without checking the node's NVIDIA driver version |
| Why it masqueraded as hardware failure | nvidia-smi works fine, GPU is allocated by K8S, pod starts normally — CUDA version check only happens at runtime initialization |

**Key concept**: The NVIDIA driver version determines the **maximum** CUDA version it can support. This relationship is NOT "whatever you installed":

- Drivers are **backward compatible** — driver 525 runs CUDA 11.x, 12.0 programs
- Drivers are **NOT forward compatible** — driver 525 cannot run programs compiled for CUDA 12.2
- Container CUDA toolkit ≤ host driver's max supported CUDA version

Driver-to-CUDA mapping (simplified):

| Driver | Max CUDA |
|--------|----------|
| 470.x | 11.4 |
| 510.x | 11.6 |
| 520.x | 12.0 |
| **525.x** | **12.0** |
| **535.x** | **12.2** |
| 545.x | 12.3 |
| 550.x | 12.4 |

## Fix

### Option A: Upgrade NVIDIA driver (recommended, root fix)

Upgrade to a driver that supports CUDA 12.2 (≥ 535):

```bash
# 1. Check current driver
nvidia-smi --query-gpu=driver_version --format=csv,noheader

# 2. Install new driver (Ubuntu/Debian)
apt update
apt install -y nvidia-driver-535

# 3. Reboot (required for driver load)
reboot

# 4. Verify
nvidia-smi | grep "CUDA Version"
```

```bash
# Or use NVIDIA's official runfile (common in production)
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/535.154.05/NVIDIA-Linux-x86_64-535.154.05.run
chmod +x NVIDIA-Linux-x86_64-535.154.05.run
./NVIDIA-Linux-x86_64-535.154.05.run --dkms -s
reboot
```

Post-upgrade verification:

```bash
nvidia-smi | grep "CUDA Version"
```

```
| NVIDIA-SMI 535.154.05    Driver Version: 535.154.05    CUDA Version: 12.2 |
```

```bash
kubectl rollout restart deployment triton-server -n inference
```

```bash
kubectl logs -n inference triton-server-xxx | grep CUDA
```

```
| CUDA   | CUDA driver version 12.2                                 |
|        | CUDA runtime version 12.2                                 |
```

Matched. Pod Running.

### Option B: Downgrade image CUDA (quick recovery, no driver change)

If the node driver can't change (other teams depend on it):

```bash
# Triton image CUDA version mapping:
# Triton 23.08 → CUDA 12.0 (compatible with driver 525)
# Triton 23.10 → CUDA 12.2 (needs driver ≥ 535)
# Triton 23.12 → CUDA 12.2 (needs driver ≥ 535)

# Use a compatible image
image: nvcr.io/nvidia/tritonserver:23.08-py3
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment triton-server -n inference
```

Verify:

```bash
kubectl logs -n inference triton-server-xxx | grep CUDA
```

```
| CUDA   | CUDA driver version 12.0                                 |
|        | CUDA runtime version 12.0                                 |
```

| Option | Action | Pros | Cons |
|--------|--------|------|------|
| A. Upgrade driver | `apt install nvidia-driver-535` | All containers benefit from newer CUDA | Needs node reboot, broader impact |
| B. Downgrade image | Use `tritonserver:23.08-py3` | Fast recovery, no node changes | Can't use newer CUDA features |

### Verification

```bash
# 1. Pod stable
kubectl get pods -n inference
# triton-server-xxx   1/1     Running

# 2. Triton health check
kubectl exec -n inference triton-server-xxx -- curl -s localhost:8000/v2/health/ready
# true

# 3. CUDA version match
kubectl logs -n inference triton-server-xxx | grep "CUDA.*version"
# Both driver and runtime versions match

# 4. Model inference test
kubectl exec -n inference triton-server-xxx -- \
  curl -s -X POST localhost:8000/v2/models/my_model/infer \
  -d '{"inputs":[...]}'
# Returns inference results
```

### Long-term prevention

```bash
# 1. Label GPU nodes with driver version
kubectl label node gpu-node-1 nvidia-driver-version=535

# 2. Use nodeSelector in deployments
# deployment.yaml
nodeSelector:
  nvidia-driver-version: "535"

# 3. CI/CD pipeline check
# Before deploying, check image CUDA version vs target node driver version

# 4. Maintain a driver version matrix
# Document: per-GPU-node driver version, supported CUDA, compatible images
```

## What I Learned

1. **nvidia-smi working ≠ CUDA programs working.** This is the most misleading pattern. nvidia-smi uses NVML, not CUDA runtime. GPU shows up fine, driver loads successfully, but CUDA programs can still refuse to run due to version mismatch. First thing when seeing CUDA errors: `nvidia-smi | grep "CUDA Version"` to see what the driver claims to support.

2. **Container CUDA toolkit cannot exceed the host driver's supported CUDA version.** It's backward compatible (CUDA 11 runs on driver 525), never forward compatible (CUDA 12.2 won't run on driver 525). Choose base images after confirming the node's driver version.

3. **Driver-CUDA compatibility requires checking a matrix, not guessing.** There's no "one driver supports everything." Every driver version has a maximum CUDA version. Before pulling the latest Triton/PyTorch/TF image, check what CUDA version it bundles and whether your nodes' drivers support it.

4. **GPU issues have three distinct layers.** Hardware (dead GPU) → Driver (not loaded/failed) → CUDA version (runtime/driver mismatch). Each layer has different symptoms: hardware fails nvidia-smi, driver problems show `nvidia.com/gpu` allocation failures in K8S, CUDA version mismatch lets the pod start but crashes the app immediately. Read the first line of the error log to determine which layer you're debugging.
