---
title: "Docker 守护进程安全配置不当——暴露的 Docker Socket 如何让宿主机沦为肉鸡"
date: 2026-06-02
weight: 100260
slug: "docker-daemon-security-misconfiguration"
tags: ["security", "docker", "container", "linux", "troubleshooting"]
categories: ["Security"]
description: "Docker 守护进程安全事件复盘——将 Docker Socket 挂载到容器中导致攻击者逃逸并控制宿主机，以及暴露的 2375 端口如何被内网扫描发现"
keywords: "docker socket 暴露, docker 权限提升, /var/run/docker.sock 安全, docker 容器逃逸宿主机, docker tcp 暴露"
draft: false
featured: true
cover:
  image: ""
  caption: "Docker 守护进程配置不当——安全事件排查"
---

# Docker 守护进程安全配置不当——暴露的 Docker Socket 如何让宿主机沦为肉鸡

## 常见搜索词

- docker.sock 暴露容器逃逸
- docker 权限提升宿主机沦陷
- /var/run/docker.sock 安全
- docker 守护进程 tcp 暴露
- docker-in-docker 安全风险

---

## 故障经过

**环境**: 单台 Docker 主机（8C/32G, Ubuntu 22.04），运行 15 个微服务容器作为 CI/CD 构建农场。Docker API 通过 TCP 2375 端口暴露（无 TLS）。

**时间**: 周二 14:30。安全扫描告警：未知容器 `evil-miner` 在 CI 构建主机上运行挖矿程序。

**发现过程**: 例行 Falco 安全扫描标记了名为 `evil-miner` 的容器，运行已知的挖矿程序。

```bash
docker ps
CONTAINER ID   IMAGE              COMMAND                  STATUS          NAMES
a1b2c3d4e5f6   xmrig/xmrig        "./xmrig --config=..."   Up 2 hours      evil-miner
f6e5d4c3b2a1   nginx:latest       "nginx -g daemon off;"   Up 14 hours     ci-web
...
```

**影响**: 宿主机和所有 15 个构建容器被入侵。CI/CD 凭据、SSH 密钥和源代码被窃取。宿主机彻底重建。

---

## 背景

CI/CD 构建农场运行在单台 Docker 主机上。为了实现"Docker-in-Docker"构建，团队将 Docker Socket（`/var/run/docker.sock`）挂载到了 CI Runner 容器中。这是众所周知的模式——也正是被利用的攻击向量。

```bash
# CI Runner 容器的启动方式
docker run -d \
  --name ci-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \  # ← Docker socket 挂载
  -v /home/ci/workspace:/workspace \
  gitlab-runner:latest
```

挂载 Docker Socket 使得容器拥有了 **Docker 守护进程的 root 级访问权限**。从容器内部可以执行任何 Docker 命令——包括启动具有完全宿主机访问权限的新容器。

此外，Docker 守护进程本身也在 TCP 2375 端口上监听，没有 TLS 加密，纯粹为了"方便"团队远程连接：

```bash
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"],
  "tls": false
}
```

攻击者扫描内部网络时发现了开放的 2375 端口。从此完全控制了 Docker 守护进程。

---

## 排查过程

### 第一步：检查攻击者如何进入

```bash
journalctl -u docker.service --since "24 hours ago" | grep -i "POST\|DELETE\|create"
```

日志显示 Docker API 调用来自两个源：
1. 内部 IP `10.0.1.50`——CI Runner 容器（通过 socket 挂载）
2. 内部 IP `10.0.1.200`——未知，通过 TCP 2375 端口的 API

攻击者通过内网扫描发现了暴露的 TCP 端口。

### 第二步：追溯攻击行为

攻击者拉取了 `xmrig/xmrig` 镜像并启动了挖矿容器。但在此之前，他们：

1. 列出所有运行中的容器 → 发现挂载了 Docker Socket 的 CI Runner
2. 利用 CI Runner 的 Docker Socket 启动了一个特权容器
3. 从特权容器挂载了宿主机文件系统
4. 窃取了 `/home/ci/.ssh/` 中的 SSH 密钥
5. 窃取了 `/home/ci/workspace/.env` 中的 CI/CD 凭据
6. 启动挖矿程序作为掩护

### 第三步：评估损失

所有 SSH 密钥、CI/CD Token 和宿主机上的环境变量全部被泄露。如果这台主机有权限访问生产集群，那些凭据也需要轮换。

---

## 根因

1. **Docker Socket 挂载到容器中**：CI Runner 挂载了 `docker.sock`，赋予该容器中的任何进程 Docker 守护进程的 root 级访问权限
2. **Docker TCP API 无 TLS 暴露**：2375 端口在 `0.0.0.0` 上开放，无认证无加密
3. **Docker API 端口无防火墙**：没有 iptables 规则或安全组限制对 2375 端口的访问
4. **允许创建特权容器**：没有策略阻止通过 API 创建特权容器

---

## 解决方案

### 紧急处置

```bash
# 停止所有未知容器
docker stop evil-miner && docker rm evil-miner

# 禁用 TCP API 访问
systemctl stop docker
sed -i 's|"tcp://0.0.0.0:2375",||' /etc/docker/daemon.json
systemctl start docker

# 防火墙封禁 2375 端口
iptables -A INPUT -p tcp --dport 2375 -j DROP
ufw deny 2375
```

### 加固 Docker 配置

```bash
{
  "hosts": ["unix:///var/run/docker.sock"],
  "tls": true,
  "icc": false,
  "live-restore": true,
  "userland-proxy": false
}
```

### 永远不要在容器中挂载 Docker Socket

替代方案：
1. **Docker-in-Docker (dind)**：在容器内运行独立的 Docker 守护进程
2. **Rootless Docker**：无 Root 权限运行 Docker
3. **Podman**：无需守护进程的 Rootless 容器引擎

如果 **必须** 挂载 Docker Socket：
1. 使用只读挂载：`:ro`
2. 使用带认证的 Docker API 代理
3. 通过仓库白名单限制可拉取的镜像

### 安全配置 Docker API

如果必须使用 TCP 访问，使用 TLS 证书认证：

```bash
openssl genrsa -aes256 -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=docker-host" -sha256 -new -key server-key.pem -out server.csr

{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2376"],
  "tls": true,
  "tlscacert": "/etc/docker/certs/ca.pem",
  "tlscert": "/etc/docker/certs/server.pem",
  "tlskey": "/etc/docker/certs/server-key.pem",
  "tlsverify": true
}
```

### 监控和审计

```yaml
# Falco 规则：检测 Docker Socket 挂载
- rule: Docker Socket Mounted in Container
  desc: 检测挂载 Docker Socket 的容器
  condition: container.mount contains "/var/run/docker.sock"
  output: "容器中挂载了 Docker Socket (user=%user.name container=%container.name)"
  priority: CRITICAL
```

```yaml
# Prometheus 告警：暴露的 Docker API
- alert: DockerAPIAccessible
  expr: probe_success{target="tcp://docker-host:2375"} == 1
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Docker API 在 2375 端口上无需 TLS 即可访问"
```

---

## 经验教训

- **`/var/run/docker.sock` 等同于 Root 权限**：在容器中挂载 Docker Socket 会使该容器拥有宿主机的 Root 访问权。没有沙箱，没有隔离——这是一个 Root 后门
- **Docker TCP API 是无密码的 Root 访问**：不加 TLS 暴露 2375 端口相当于把 Root 密码贴在公网上。谁发现谁就能控制宿主机
- **CI/CD Runner 永远不应有 Socket 访问权限**：使用 dind、Rootless Docker 或 Podman。构建容器不需要守护进程访问
- **Docker-in-Docker 模式是危险的默认选择**：Socket 挂载的便利性导致了无数安全事件。这不是 CI/CD 的必需品——而是安全风险
- **定期审计容器挂载**：`docker inspect` 显示所有挂载。每周扫描 `docker.sock` 挂载

---

## 总结

攻击链路：

```
Docker 主机配置了 TCP API 在 2375 端口（无 TLS）
→ 攻击者扫描内部网络，发现开放端口
→ 连接 Docker API——无需密码
→ 列出所有容器，发现挂载了 docker.sock 的 CI Runner
→ 启动带宿主机文件系统挂载的特权容器
→ 窃取 SSH 密钥和 CI/CD 凭据
→ 启动挖矿容器作为掩护
→ 15 个构建容器被入侵，宿主机完全沦陷
```

恢复：宿主机重建 + 凭据轮换，6 小时。修复：30 分钟的守护进程重新配置。Docker Socket 很强大——像对待 Root 密码一样对待它。因为它就是 Root 密码。
