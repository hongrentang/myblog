---
title: "Ceph OSD Down 导致存储集群降级—一次磁盘故障引发的血案"
date: 2026-05-19
weight: 100050
slug: "ceph-osd-down-storage-cluster-degraded"
tags: ["ceph", "storage", "troubleshooting"]
categories: ["存储"]
description: "Ceph OSD 宕机导致存储集群降级的完整排查过程，从误判网络到定位磁盘坏道"
keywords: "ceph osd down, ceph pg degraded, ceph存储集群故障, osd宕机排查, ceph数据恢复"
draft: false
featured: true
cover:
  image: "/images/ceph-osd-down-banner.svg"
  caption: "Ceph OSD Down 排查记录"
---

# Ceph OSD Down 导致存储集群降级

## 问题背景

这事儿发生在周二下午两点多。我们 Ceph 集群跑了一年多一直挺稳的，三个 MON 节点、12 个 OSD 节点（每个节点 6 块 HDD），整体 72 个 OSD。存储池是三副本，平时 IO 延迟大概 5-10ms。

那天我正在跟另一个同事对需求，突然告警叮叮咚咚响起来——Ceph 集群状态从 HEALTH_OK 变成 HEALTH_WARN，紧接着又变成 HEALTH_ERR。当时心里咯噔一下。

```bash
# 收到告警时的输出
cluster:
    id:     a3f8c2d1-...
    health: HEALTH_ERR
            3 osds down
            47 pgs degraded
            12 pgs stuck stale
            1 pool has unfound objects
```

**影响**：三副本意味着最多能坏两个副本不影响数据，但 3 个 OSD 同时 down，如果有 PG 刚好三个副本都在这些 OSD 上——那数据就丢了。更重要的是，依赖这个存储的 Kafka、ES 集群都在报 IO 超时。

## 排查过程

### 错误尝试 1：第一反应是网络问题

看到 3 个 OSD down，我第一反应是——是不是交换机出问题了？之前遇到过 OSD 之间心跳超时导致误判 down 的情况。

```bash
# 先看这 3 个 OSD 在哪些节点上
ceph osd tree | grep -E "down|osd\."
```

```
-3       10.00  host storage-node-05
 15    1.90   down        1.0000  osd.15
 16    1.90   down        1.0000  osd.16
 42    1.90   down        1.0000  osd.42
```

三个 down 的 OSD 全都在 **同一个节点** storage-node-05 上。那大概率不是网络问题——如果是交换机故障，应该是跨节点的多个 OSD 同时 down。

### 错误尝试 2：以为是 OSD 进程 hang 住

SSH 到 storage-node-05：

```bash
ssh storage-node-05
systemctl status ceph-osd@15
```

```
● ceph-osd@15.service - Ceph OSD 15
   Loaded: loaded /usr/lib/systemd/system/ceph-osd@.service
   Active: active (running) since Mon 2026-05-18 22:00:00 CST
  Process: 12345 ExecStart=/usr/bin/ceph-osd -f -i 15 (code=killed, signal=KILL)
 Main PID: 23456 (ceph-osd)
   Status: "HEALTH_OK"
```

OSD 进程竟然显示 active running？那为什么 ceph -s 说 down？这时候我犯了个错——只看 systemctl 状态没看日志。

后来同事说了一句"你看 journalctl 了吗"，我才反应过来：

```bash
journalctl -u ceph-osd@15 --since "2 hours ago" | tail -50
```

关键输出：

```
May 19 14:15:23 storage-node-05 ceph-osd[12345]: ** ERROR: osd.15 has been marked down by mon.a
May 19 14:15:23 storage-node-05 ceph-osd[12345]: ** or its heartbeat packets are not being received
May 19 14:15:23 storage-node-05 ceph-osd[12345]: ** This could indicate a hardware or kernel issue
May 19 14:15:25 storage-node-05 ceph-osd[12345]: osd.15 192.168.1.105:6800/12345 --> ** MESSAGE DELAY: 380.5s **
May 19 14:15:25 storage-node-05 ceph-osd[12345]: osd.15 192.168.1.105:6800/12345 --> heartbeat timeout from osd.42
May 19 14:15:25 storage-node-05 ceph-osd[12345]: osd.15: ** SLOW OSD HEARTBEAT ** detected
```

**踩坑点**：systemctl 显示 active 不代表 OSD 正常工作。MON 把 OSD 标记为 down 后，进程虽然还在跑但已经不参与数据服务了。只看 systemctl 浪费了 10 分钟。

### 错误尝试 3：以为是内核 OOM 杀了进程

看到 "killed, signal=KILL" 我第一反应是 OOM killer 把进程干掉了。检查了 dmesg：

```bash
dmesg | grep -i oom
dmesg | grep -i kill
```

没有任何 OOM 记录。内存也够：

```bash
free -h
```

```
              total        used        free      shared  buff/cache
Mem:          125Gi        48Gi        62Gi        2.3Gi        15Gi
```

内存没问题。那为什么 process 被 kill 了？

### 真正的排查开始

回头看 journalctl 的完整日志，发现了一个被我忽略的细节：

```
May 19 14:10:23 storage-node-05 ceph-osd[12345]: /var/lib/ceph/osd/ceph-15/: ** read error ** at offset 0x7b4a00000, length 4096
May 19 14:10:23 storage-node-05 ceph-osd[12345]: ** ERROR: bluefs_fsync: fsync on /var/lib/ceph/osd/ceph-15//block.wal failed: Input/output error
May 19 14:10:23 storage-node-05 ceph-osd[12345]: ** Fatal: bluefs: during fsync, abort
```

**Input/output error**——这不是软件问题，是磁盘坏了。

```bash
# 检查 OSD 使用的磁盘
mount | grep ceph-15
```

```
/dev/sdc1 on /var/lib/ceph/osd/ceph-15 type xfs (rw,noatime)
```

直接检查磁盘健康状态：

```bash
smartctl -a /dev/sdc | grep -E "Reallocated|Pending|Offline|UDMA|Current_Pending"
```

```
  5 Reallocated_Sector_Ct   198   198   140    -    583
197 Current_Pending_Sector   252   252   000    -    1265
198 Offline_Uncorrectable    252   252   000    -    943
```

583 个重新分配扇区、1265 个待分配扇区——这盘已经废了。

但问题是，为什么三个 OSD 同时 down？检查另外两个 OSD 的磁盘：

```bash
mount | grep -E "ceph-16|ceph-42"
```

```
/dev/sdd1 on /var/lib/ceph/osd/ceph-16 type xfs (rw,noatime)
/dev/sde1 on /var/lib/ceph/osd/ceph-42 type xfs (rw,noatime)
```

smartctl 查了一下 sdd 和 sde，发现它们也有坏道，但没 sdc 那么严重。进一步调查发现——storage-node-05 这三年没换过磁盘，这批 HDD 都是同一批次采购的。不是一块盘坏了拖累其他盘，而是三块盘都在独立地走向失效，sdc 只是最严重的那块。

```bash
# 看三块盘的 IO 延迟
iostat -x 1 5 | grep -E "sdc|sdd|sde|await"
```

```
Device     r/s     w/s     rkB/s     wkB/s  await  svctm  %util
sdc       0.5     0.2       2.0       1.0  8500    5200   99.8
sdd      12.3     8.1     512.0     340.0  3200    1800   68.5
sde      18.5    12.2     768.0     512.0  1800     950   55.2
```

sdc 的 await 高达 8500ms——每次 IO 要等 8.5 秒，这盘基本废了。

## 根因分析

这事说复杂也复杂，说简单也简单：

1. **直接原因**：storage-node-05 上的 `/dev/sdc` 出现大量坏道，导致 OSD.15 的 WAL 写入失败，进程被 abort
2. **并行失效**：sdd、sde 本身也有坏道（同批次、同年代），不是被 sdc 拖累的。三块盘各自 IO 延迟飙升，OSD 心跳超时，被 MON 一并标记 down
3. **根本原因**：这批 HDD 已经跑了快 4 年，超出了厂商建议的 3-5 年更换周期。而且没有做定期 smartctl 巡检，坏道累积到一定程度才被发现

**最大的误判**：前 15 分钟我一直在排查网络、排查进程、排查 OOM，就是没看磁盘。如果一开始就 `journalctl -u ceph-osd@15 | grep "error"`，5 分钟就能定位到盘坏了。

## 解决方案

### 第一步：踢出坏盘（热修复）

```bash
# 先把坏盘上的 OSD 踢出集群
ceph osd out osd.15
ceph osd out osd.16
ceph osd out osd.42
```

然后等 PG 自动重新平衡（这个花了大概 40 分钟，数据量比较大）。

```bash
# 监控 PG 恢复状态
ceph -w | grep -E "recover|degraded|active+clean"
```

### 第二步：停止坏 OSD，换盘

```bash
systemctl stop ceph-osd@15
systemctl stop ceph-osd@16
systemctl stop ceph-osd@42

# 从 CRUSH map 中移除
ceph osd crush remove osd.15
ceph osd crush remove osd.16
ceph osd crush remove osd.42

# 删除 OSD 认证
ceph auth del osd.15
ceph auth del osd.16
ceph auth del osd.42

# 从集群中删除
ceph osd rm osd.15
ceph osd rm osd.16
ceph osd rm osd.42
```

### 第三步：换上新盘重新加入

换了新盘后，重新创建 OSD：

```bash
# 格式化新盘
mkfs.xfs /dev/sdc1 -f
mkfs.xfs /dev/sdd1 -f
mkfs.xfs /dev/sde1 -f

# 重新创建 OSD
ceph-volume lvm create --data /dev/sdc1 --osd-id 15
ceph-volume lvm create --data /dev/sdd1 --osd-id 16
ceph-volume lvm create --data /dev/sde1 --osd-id 42
```

### 验证恢复

```bash
# 检查集群状态
ceph -s
```

```
cluster:
    id:     a3f8c2d1-...
    health: HEALTH_OK

services:
    mon: 3 daemons, quorum a,b,c
    osd: 72 osds: 72 up, 72 in

data:
    pools:   12 pools, 1024 pgs
    objects: 2.3M objects, 8.7 TiB
    usage:   26 TiB used, 42 TiB / 68 TiB avail
    pgs:     1024 active+clean
```

### 长期修复

1. **部署 smartctl 巡检脚本**，每周跑一次，坏道超过阈值自动告警
2. **设置 OSD 淘汰机制** —— 当 OSD 心跳延迟超过 100ms 时自动触发告警而不是等 MON 强制踢掉
3. **同批次磁盘统一更换**——不能一块坏了再换一块，其他同批次的盘大概率也快了
4. **增加 WAL/DB 分区到 NVME**——之前这个节点没有用 NVME 做 WAL 加速，这次整改补上了

```bash
# 加的告警规则（Prometheus）
# ceph_osd_heartbeat_latency_seconds > 0.1
# ceph_osd_stat_bytes_used / ceph_osd_stat_bytes > 0.85
```

## 教训总结

1. **journalctl 是定位 OSD 问题的第一站**，不是 systemctl status。OSD active 不代表 OSD 正常
2. **同一个节点多个 OSD down，先查磁盘，别查网络**。同节点多 OSD 同时故障大概率是硬件问题
3. **HDD 超过 3 年就要重点关注 smartctl 指标**。Reallocated_Sector_Ct > 100 就该考虑换盘了，等到 IO error 就晚了
4. **同批次磁盘会集中失效**：一块盘出问题时，同批次的其他盘大概率也快到寿命了。与其等它坏了再换，不如主动批量替换
5. **备份永远最好使**——三副本不是万能药。如果是同一个节点上的三块盘同时坏，PG 的三副本可能都在这个节点上。定期检查 CRUSH 分布规则是必要的
