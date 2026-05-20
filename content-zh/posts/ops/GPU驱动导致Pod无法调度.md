---
title: "GPU Pod 卡在 ContainerCreating——NVIDIA 容器运行时配置丢失排查"
date: 2026-05-20
weight: 100100
slug: "gpu-pod-containercreating-nvidia-runtime"
tags: ["gpu", "nvidia", "kubernetes", "troubleshooting"]
categories: ["AI/ML"]
description: "NVIDIA 容器运行时配置丢失导致 GPU Pod 无法启动的完整排查，从误判驱动问题到定位 containerd 配置"
keywords: "gpu pod containercreating, nvidia-container-runtime, containerd配置, k8s gpu 无法调度, nvidia runtime not found"
draft: false
featured: true
cover:
  image: "/images/gpu-containerruntime-banner.svg"
  caption: "GPU Pod ContainerCreating 排查"
---

# GPU Pod 卡在 ContainerCreating——NVIDIA 容器运行时配置丢失排查

## 问题现象

节点重启后，所有 GPU Pod 都卡在了 `ContainerCreating`。

```bash
kubectl get pods -n inference
```

```
NAME                          READY   STATUS              RESTARTS   AGE
triton-server-v1              0/1     ContainerCreating   0          23m
triton-server-v2              0/1     ContainerCreating   0          23m
triton-server-v3              0/1     ContainerCreating   0          22m
```

查看其中一个 Pod 的详细信息：

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

关键错误：**unknown runtime "nvidia"**。containerd 不认识 `nvidia` 这个运行时。

**影响**：所有推理服务不可用，GPU 节点上有 3 个 Triton 实例和 2 个 vLLM 实例全部宕了。

## 排查过程

### 错误尝试 1：以为是驱动问题

看到 `unknown runtime "nvidia"`，我第一反应是 NVIDIA 驱动坏了。先确认驱动状态：

```bash
# SSH 到 GPU 节点
ssh gpu-node-01

# 检查 GPU 状态
nvidia-smi
```

```
Tue May 20 14:30:00 2026
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 535.154.05   Driver Version: 535.154.05   CUDA Version: 12.2     |
+-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
|===============================+======================+======================|
|   0  Tesla T4            On   | 00000000:00:1E.0 Off |                    0 |
|   1  Tesla T4            On   | 00000000:00:1F.0 Off |                    0 |
+-------------------------------+----------------------+----------------------+
```

驱动正常，GPU 都在线。那问题不是驱动。

试试手动跑 GPU 容器：

```bash
docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi
```

```
docker: Error response from daemon: unknown runtime "nvidia".
```

Docker 也报同样的错。这不只是 K8S 的问题，是整个节点的容器运行时都不认识 `nvidia` 这个 runtime。

**踩坑点**：`nvidia-smi` 正常不代表 GPU 容器可用。nvidia-smi 走的是内核驱动，而 GPU 容器还需要 `nvidia-container-runtime` 这一层。驱动好 + runtime 挂 = 还是跑不了。

### 错误尝试 2：看 containerd 配置，但看错地方

既然 `unknown runtime "nvidia"`，那应该是 containerd 的配置里没有注册 nvidia runtime。打开 containerd 配置：

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

确实没有 `nvidia` 运行时。但问题来了——我第一反应是自己手写配置，然后忘了备份原文件，写错了格式：

```bash
# 错误示范——手写配置容易出格式错误
vim /etc/containerd/config.toml
```

加了一段 nvidia runtime 配置，结果 containerd 重启失败：

```bash
systemctl restart containerd
systemctl status containerd
```

```
● containerd.service - containerd container runtime
   Loaded: loaded (/usr/lib/systemd/system/containerd.service; enabled;)
   Active: failed (Result: exit-code) since Tue 2026-05-20 14:35:00 CST
```

配置文件格式写错了。

```bash
# 回退并检查配置语法
containerd config dump > /dev/null 2>&1
# 有报错说明配置不对
```

**踩坑点**：containerd 的 config.toml 格式非常敏感，缩进层级错一个 TOML 键路径就失效。不要手写，用 `nvidia-ctk` 自动生成。

### 错误尝试 3：重新安装 NVIDIA 驱动

"runtime 丢了？那重装驱动和 runtime 吧。"

```bash
# 卸载现有驱动
apt-get purge --remove nvidia-*
apt-get autoremove

# 重装驱动
apt-get install nvidia-driver-535
```

重启后：

```bash
nvidia-smi
```

正常。但是：

```bash
cat /etc/containerd/config.toml | grep nvidia
```

还是没有。重装驱动不会帮你配 containerd。

**踩坑点**：NVIDIA 驱动安装包包含 `nvidia-container-runtime` 但**不包含** `nvidia-container-toolkit` 的 containerd 配置步骤。驱动重装一百遍也不会自动加到 containerd 配置里。`nvidia-ctk` 这个工具才是做配置的。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | containerd 的 config.toml 中缺少 nvidia runtime 定义 |
| 触发原因 | 节点重启后系统更新覆盖了 containerd 配置，或配置丢失 |
| 根本原因 | NVIDIA GPU 节点的容器运行时配置没有被持久化到配置管理 |

**nvidia runtime 的工作链路**：

```
Pod → containerd → nvidia-container-runtime → nvidia-container-toolkit → nvidia-smi (驱动)
```

每一层都依赖下一层注册好。Pod 报 `unknown runtime "nvidia"`，说明 containerd 这层就没配好。

## 解决方案

### 方案 A：使用 nvidia-ctk 自动配置（推荐，最快最稳）

```bash
# 1. 检查 nvidia-ctk 是否安装
which nvidia-ctk
# 如果没有安装
apt-get install nvidia-container-toolkit

# 2. 自动配置 containerd（一行命令搞定）
nvidia-ctk runtime configure --runtime=containerd
```

```
INFO[0000] Loading config from /etc/containerd/config.toml
INFO[0000] Successfully loaded config
INFO[0000] Writing config to /etc/containerd/config.toml
INFO[0000] Successfully written config
```

```bash
# 3. 重启 containerd
systemctl restart containerd

# 4. 确认配置生效
cat /etc/containerd/config.toml | grep -A 5 "nvidia"
```

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
```

```bash
# 5. 验证 GPU 容器可运行
docker run --rm --gpus all nvidia/cuda:12.2-base nvidia-smi
# 正常输出 GPU 信息
```

### 方案 B：手动验证 Pod 恢复

```bash
# 重新创建 GPU Pod（runtime 丢失期间 Pod 不会自动恢复）
kubectl delete pod triton-server-v1 -n inference
# 或者删除所有 stuck 的 Pod
kubectl delete pods -n inference --field-selector=status.phase=Running
```

新创建的 Pod 应该正常启动：

```bash
kubectl get pods -n inference
```

```
NAME                          READY   STATUS    RESTARTS   AGE
triton-server-v1              1/1     Running   0          30s
triton-server-v2              1/1     Running   0          28s
triton-server-v3              1/1     Running   0          26s
```

✅ **恢复验证**：

```bash
# 1. 检查 Pod 日志确认 GPU 可用
kubectl logs triton-server-v1 -n inference | tail -5
# 预期：正常加载模型，没有 CUDA 报错

# 2. 检查 GPU 显存占用
ssh gpu-node-01
nvidia-smi
# 预期：GPU 0/1 有进程占用显存

# 3. 调一下推理 API 确认服务正常
curl http://triton-server:8000/v2/health/ready
# 预期：返回 true
```

### 方案 C：配置持久化（防止复发）

```bash
# 将 containerd 配置纳入配置管理（Ansible / Salt / 自定义脚本）
# 关键：确保节点重建或重启后 nvidia runtime 配置不会丢

# 如果是新节点装机的场景，纳入初始化脚本：
#!/bin/bash
# GPU 节点初始化脚本片段
apt-get install -y nvidia-driver-535 nvidia-container-toolkit
nvidia-ctk runtime configure --runtime=containerd
systemctl restart containerd
```

**三种方式对比**：

| 方式 | 适用场景 | 风险 |
|------|----------|------|
| A. nvidia-ctk 配置 | 已安装 toolkit 的节点 | 低，官方工具 |
| B. 手动编辑 config.toml | 无 nvidia-ctk 的老系统 | 中，容易写错格式 |
| C. 配置管理持久化 | 生产环境 | 低，但需 CM 系统 |

**如何判断节点是否已安装 nvidia-container-toolkit**：

```bash
dpkg -l | grep nvidia-container-toolkit
# 或者
rpm -qa | grep nvidia-container-toolkit
```

如果没有输出，需要先安装：

```bash
# Ubuntu/Debian
apt-get install -y nvidia-container-toolkit

# CentOS/RHEL
yum install -y nvidia-container-toolkit
```

## 长期预防

```bash
# 1. 节点初始化脚本中加入 runtime 配置检查
# 在 /etc/containerd/ 下保留备份
cp /etc/containerd/config.toml /etc/containerd/config.toml.bak

# 2. 配置 containerd 健康检查
# 定期检查 nvidia runtime 是否存在
grep -q "nvidia" /etc/containerd/config.toml || echo "ALERT: nvidia runtime not configured"

# 3. GPU 节点重建 SOP 中加入这一步
# 不要只写"安装驱动"，要明确写"nvidia-ctk runtime configure"
```

## 教训总结

1. **`nvidia-smi` 正常 ≠ GPU 容器能跑**。nvidia-smi 走内核驱动，容器还需要 `nvidia-container-runtime` 这一层。排查 GPU 问题要分层：驱动 → runtime → containerd 配置 → Pod。

2. **containerd 的 config.toml 不要手写**。缩进错一个 TOML 层级整个 containerd 就起不来。用 `nvidia-ctk runtime configure` 一行搞定。

3. **重装驱动不会自动配 containerd**。驱动包安装 `nvidia-container-runtime` 可执行文件但不会帮你注册到 containerd。跑完驱动安装必须加一步 `nvidia-ctk runtime configure`。

4. **节点重启后 GPU 配置丢失是个常见坑**。系统更新或配置管理工具可能覆盖 containerd 配置。建议把 config.toml 纳入版本控制或配置管理系统，节点重建后能自动恢复。

5. **排查 K8S GPU 问题的第一组命令**：`kubectl describe pod` 看 Events → 确认 `unknown runtime` → SSH 到节点看 containerd 配置。跳过"重装驱动"这个最大的坑。
