---
title: "GPU Pod Stuck in ContainerCreating — NVIDIA Container Runtime Config Lost"
date: 2026-05-20
weight: 100100
slug: "gpu-pod-containercreating-nvidia-runtime"
tags: ["gpu", "nvidia", "kubernetes", "troubleshooting"]
categories: ["AI/ML"]
description: "Full postmortem of GPU pods stuck in ContainerCreating — from misdiagnosing drivers to finding missing nvidia runtime in containerd config"
keywords: "gpu pod containercreating, nvidia-container-runtime, containerd config, k8s gpu scheduling, nvidia runtime not found"
draft: false
featured: true
cover:
  image: "/images/gpu-containerruntime-banner.svg"
  caption: "GPU Pod ContainerCreating Troubleshooting"
---

# GPU Pod Stuck in ContainerCreating — NVIDIA Container Runtime Config Lost

## Symptoms

After a node reboot, all GPU pods got stuck in `ContainerCreating`.

```bash
kubectl get pods -n inference
```

```
NAME                          READY   STATUS              RESTARTS   AGE
triton-server-v1              0/1     ContainerCreating   0          23m
triton-server-v2              0/1     ContainerCreating   0          23m
```

Check one pod's details:

```bash
kubectl describe pod triton-server-v1 -n inference
```

```
Events:
  Type     Reason                  Age   From               Message
  ----     ------                  ----  ----               -------
  Normal   Scheduled               23m   default-scheduler  Successfully assigned to gpu-node-01
  Warning  FailedCreatePodSandBox  23m   kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create containerd container: unknown runtime "nvidia"
```

Key error: **unknown runtime "nvidia"**. containerd doesn't know about the `nvidia` runtime.

**Impact**: All inference services down — 3 Triton instances and 2 vLLM instances on this node.

## Investigation

### Wrong Turn 1: Blamed the GPU Driver

`unknown runtime "nvidia"` — I assumed the NVIDIA driver was broken.

```bash
ssh gpu-node-01
nvidia-smi
```

```
Tue May 20 14:30:00 2026
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.154.05   Driver Version: 535.154.05   CUDA Version: 12.2     |
+-------------------------------+----------------------+----------------------+
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
|   1  Tesla T4            On   | 00000000:00:1F.0 Off |                    0 |
+-------------------------------+----------------------+----------------------+
```

Driver is fine. Both GPUs online.

```bash
# Try running a GPU container manually
docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi
```

```
docker: Error response from daemon: unknown runtime "nvidia".
```

Same error from Docker. This isn't just a K8S issue — the whole node's container runtime doesn't know about `nvidia`.

**Lesson**: `nvidia-smi` working doesn't mean GPU containers can run. nvidia-smi uses the kernel driver directly, but containers need `nvidia-container-runtime` on top.

### Wrong Turn 2: Edited containerd Config by Hand

I opened containerd's config, expecting to add the nvidia runtime:

```bash
cat /etc/containerd/config.toml
```

```
version = 2
root = "/var/lib/containerd"

[plugins]
  [plugins."io.containerd.grpc.v1.cri"]
    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
```

No nvidia runtime defined. I made my first mistake — **editing TOML by hand without a backup**:

```bash
vim /etc/containerd/config.toml
```

I broke the format. containerd wouldn't restart:

```bash
systemctl restart containerd
systemctl status containerd
```

```
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled;)
   Active: failed (Result: exit-code) since Tue 2026-05-20 14:35:00 CST
```

**Lesson**: containerd's config.toml is notoriously format-sensitive. Don't edit by hand — use `nvidia-ctk` instead.

### Wrong Turn 3: Reinstalled the NVIDIA Driver

"Runtime lost? Let me reinstall the driver."

```bash
apt-get purge --remove nvidia-*
apt-get autoremove
apt-get install nvidia-driver-535
```

Reboot. `nvidia-smi` works. But:

```bash
cat /etc/containerd/config.toml | grep nvidia
```

Still no nvidia config. **Reinstalling the driver doesn't touch containerd config.**

**Lesson**: The NVIDIA driver package installs `nvidia-container-runtime` but **does not** register it with containerd. You need `nvidia-ctk` for that step. Reinstalling the driver a hundred times won't fix this.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct cause | containerd config.toml missing the nvidia runtime definition |
| Trigger | Node reboot or system update overwrote the containerd config |
| Root cause | NVIDIA GPU node runtime config wasn't persisted to configuration management |

**The nvidia container chain**:

```
Pod → containerd → nvidia-container-runtime → nvidia-container-toolkit → nvidia-smi (driver)
```

Each layer depends on the previous one being registered. `unknown runtime "nvidia"` means the chain breaks at containerd.

## Solutions

### Option A: Use nvidia-ctk (Recommended)

```bash
# 1. Check if nvidia-ctk is installed
which nvidia-ctk
# If not:
apt-get install nvidia-container-toolkit

# 2. Auto-configure containerd
nvidia-ctk runtime configure --runtime=containerd
```

```
INFO[0000] Loading config from /etc/containerd/config.toml
INFO[0000] Successfully loaded config
```

```bash
# 3. Restart containerd
systemctl restart containerd

# 4. Verify config
cat /etc/containerd/config.toml | grep -A 5 "nvidia"
```

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
```

```bash
# 5. Test
docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi
# Expected: GPU info output
```

### Option B: Verify Pod Recovery

```bash
# Stuck pods won't auto-recover — recreate them
kubectl delete pod triton-server-v1 -n inference
# Or delete all stuck GPU pods
kubectl delete pods -n inference --field-selector=status.phase=Running
```

New pods should start normally:

```bash
kubectl get pods -n inference
```

```
NAME                          READY   STATUS    RESTARTS   AGE
triton-server-v1              1/1     Running   0          30s
```

✅ **Recovery verification**:

```bash
# 1. Check pod logs for CUDA availability
kubectl logs triton-server-v1 -n inference | tail -5

# 2. Check GPU memory usage
ssh gpu-node-01
nvidia-smi
# Expected: processes consuming GPU memory

# 3. Test inference endpoint
curl http://triton-server:8000/v2/health/ready
# Expected: true
```

### Option C: Config Persistence (Prevent Recurrence)

```bash
# Add to node bootstrap script:
apt-get install -y nvidia-driver-535 nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=containerd
systemctl restart containerd
```

**Option comparison**:

| Method | Scenario | Risk |
|--------|----------|------|
| A. nvidia-ctk configure | Node with toolkit installed | Low, official tool |
| B. Manual config.toml edit | Legacy system without nvidia-ctk | Medium, format errors |
| C. Config management | Production | Low, requires CM system |

## Long-Term Prevention

```bash
# 1. Node bootstrap checklist must include nvidia-ctk step
# Not just "install driver" but explicitly "nvidia-ctk runtime configure"

# 2. Periodic check for nvidia runtime in containerd config
grep -q "nvidia" /etc/containerd/config.toml || echo "ALERT: nvidia runtime not configured"

# 3. Backup containerd config
cp /etc/containerd/config.toml /etc/containerd/config.toml.bak
```

## What I Learned

1. **`nvidia-smi` OK ≠ GPU containers OK**. nvidia-smi uses the kernel driver directly. Containers need `nvidia-container-runtime` registered with containerd. Debug layers: driver → runtime → containerd config → Pod.

2. **Never hand-edit containerd's config.toml**. TOML indent hierarchy is critical — one wrong space and containerd won't start. Use `nvidia-ctk runtime configure` — it's one command and it works.

3. **Reinstalling the driver won't fix runtime config**. The driver installs the binary but doesn't register it. The `nvidia-ctk` step is separate and mandatory.

4. **Node reboots expose runtime config fragility**. System updates or config management tools can overwrite containerd config. Track it in version control.

5. **First commands for K8S GPU issues**: `kubectl describe pod` for Events → identify `unknown runtime` → SSH to node → check containerd config. Skip the "reinstall driver" trap.
