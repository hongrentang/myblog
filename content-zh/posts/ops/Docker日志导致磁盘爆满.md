---
title: "Docker 日志导致磁盘爆满——从告警到根治的完整排查"
date: 2026-05-19
weight: 100060
slug: "docker-log-disk-full-troubleshooting"
tags: ["docker", "linux", "troubleshooting"]
categories: ["运维"]
description: "Docker 容器日志导致磁盘爆满的完整排查与根治方案，从紧急止血到长期预防"
keywords: "docker日志清理, docker磁盘爆满, json.log清理, docker日志轮转, daemon.json配置"
draft: false
featured: true
cover:
  image: "/images/docker-log-disk-full-banner.svg"
  caption: "Docker 日志磁盘爆满排查"
---

# Docker 日志导致磁盘爆满——从告警到根治的完整排查

## 一、问题现象

磁盘告警响了——使用率 95%。

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G   47G  3.0G  94% /
```

根分区快写满了。再拖下去，数据库会写不进去、日志会打不出来、SSH 都可能登不上。

## 二、排查过程

### 2.1 定位大目录

```bash
# 从根目录逐级往下查，找到哪在吃磁盘
du -sh /* | sort -rh | head -5
```

```
32G   /var
8.5G  /usr
7.2G  /home
```

`/var` 吃了 32G，钻进去看。

```bash
# 查 /var 下哪个目录最大
du -sh /var/* | sort -rh | head -5
```

```
30G   /var/lib/docker
```

Docker 目录占了 30G。

### 2.2 定位容器日志

```bash
# Docker 容器日志全在 containers 目录下，按大小排
du -sh /var/lib/docker/containers/*/*.log | sort -rh | head -5
```

```
8.2G  /var/lib/docker/containers/abc123/abc123-json.log
5.1G  /var/lib/docker/containers/def456/def456-json.log
3.8G  /var/lib/docker/containers/ghi789/ghi789-json.log
```

**核心发现**：好几个容器的 `json.log` 文件都到 GB 级别了。Docker 默认的 logging driver 是 `json-file`，**日志文件不会自动轮转**，会一直增长直到撑爆磁盘。

### 2.3 确认日志内容（可选）

```bash
# 看这个容器在疯狂打什么
tail -50 /var/lib/docker/containers/abc123/abc123-json.log | jq '.log' | head -10
```

```
"2026-05-19 14:00:01 ERROR Failed to connect to database..."
"2026-05-19 14:00:02 ERROR Retrying connection..."
"2026-05-19 14:00:03 ERROR Failed to connect to database..."
```

应用在疯狂重试数据库连接，每秒钟刷好几条 ERROR。日志文件就这样从 MB 涨到了 GB。

⚠️ **踩坑预警**：走到这一步，**千万不要直接 `rm *.log`**。因为文件还被 Docker 进程持有，rm 只是删了文件名，进程还在往那个 inode 里写，磁盘空间不会释放。正确的做法见下方方案 A。

## 三、根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | 应用不断刷 ERROR 日志 |
| 表层原因 | Docker json-file 驱动默认不轮转 |
| 根本原因 | 没有配置日志大小上限，也没有日志采集归档机制 |

一句话：**Docker 本身不会帮你管日志大小，不配置就会爆。**

## 四、解决方案

### 方案 A：紧急清理（立刻止血）

```bash
# 1. 查看当前日志文件大小
ls -lh /var/lib/docker/containers/abc123/abc123-json.log

# 2. 截断日志文件（不是删除！）
truncate -s 0 /var/lib/docker/containers/abc123/abc123-json.log

# 3. 验证磁盘是否释放
df -h /
```

✅ **成功标准**：`df -h` 显示使用率明显下降。

```bash
# 如果容器多，一行全清理
truncate -s 0 /var/lib/docker/containers/*/*.log
```

**为什么用 truncate 不用 rm**：

| 操作 | 结果 |
|------|------|
| `rm *.log` | ❌ 不释放空间（进程持有 fd） |
| `truncate -s 0 file` | ✅ 立即释放，无需重启容器 |
| `cat /dev/null > file` | ✅ 同上，等价 |

### 方案 B：单容器限制日志大小

启动容器时指定：

```bash
# 限制日志文件最大 100M，最多保留 3 个
docker run -d \
  --log-opt max-size=100m \
  --log-opt max-file=3 \
  --name my-app \
  nginx:latest
```

✅ **验证方法**：

```bash
# 查看容器日志驱动配置
docker inspect my-app --format '{{.HostConfig.LogConfig}}'
```

预期输出包含 `max-size:100m max-file:3`。

### 方案 C：全局配置（推荐，一劳永逸）

```bash
# 1. 编辑 Docker daemon 配置
vim /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

```bash
# 2. 重载配置（不中断运行中容器）
systemctl reload docker

# 3. 验证配置生效
docker info | grep -A 5 "Logging Driver"
```

```
 Logging Driver: json-file
 Log:
  max-file: 3
  max-size: 100m
```

⚠️ **踩坑预警**：`systemctl reload docker` **只影响新启动的容器**。已有容器还是用旧配置，需要重建才能生效：

```bash
# 对已有容器，必须重建
docker-compose down && docker-compose up -d
```

### 方案 D：切换 local 日志驱动（生产环境推荐）

Docker 的 `local` 驱动内置轮转，性能比 `json-file` 好，格式是二进制文件而非 JSON：

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

**四种方案对比**：

| 方案 | 适用场景 | 生效范围 | 是否重启 |
|------|----------|----------|----------|
| A. 紧急清理 | 磁盘马上爆满 | 单次 | 否 |
| B. 单容器限制 | 临时部署/测试 | 单容器 | 需重建 |
| C. 全局 json-file | 可控环境 | 全局 | 新容器自动 |
| D. 全局 local | 生产环境 | 全局 | 新容器自动 |

## 五、验证方法

```bash
# 1. 确认日志驱动配置已生效
docker info --format '{{.LoggingDriver}}'
# 预期输出: json-file 或 local

# 2. 检查日志文件是否还在增长
ls -lh /var/lib/docker/containers/*/*.log

# 3. 启动一个新容器，验证日志轮转
docker run -d --name log-test alpine sh -c "while true; do echo 'test'; done"
sleep 5
# 检查日志文件是否超过 max-size
ls -lh /var/lib/docker/containers/$(docker ps -q --filter name=log-test)/*.log
docker rm -f log-test
```

✅ **最终验证标准**：
- `docker info` 显示预期的 Logging Driver 和配置
- 新容器日志文件不超过 100M
- 磁盘使用率稳定在 80% 以下

## 六、长期预防

```bash
# 1. 监控告警：磁盘使用率超过 80% 自动告警
# Prometheus: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.2

# 2. 每周巡检：找出超过 500M 的容器日志
find /var/lib/docker/containers/ -name "*.log" -size +500M -exec ls -lh {} \;

# 3. 生产环境对接外部日志系统（ELK / Loki）
# 日志采集后本地可以不保留，或者只保留最近 N 小时
```

**维护 checklist**：
- [ ] 新建服务必须指定 `--log-opt max-size`
- [ ] 每季度检查 `daemon.json` 是否被覆盖（部分管理工具会重写）
- [ ] 确保日志采集系统有磁盘告警（ELK 本身也会爆）
