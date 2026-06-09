---
title: "磁盘 IO 延迟飙升与 iowait 异常排查 — 记一次由 SAS 线缆故障引发的生产事故"
date: 2026-06-09
weight: 100440
slug: "disk-io-latency-iowait-spike"
tags: ["storage", "troubleshooting", "linux", "disk", "performance"]
categories: ["Troubleshooting"]
description: "一次磁盘 IO 延迟事故复盘 — JBOD 扩展柜中故障的 SAS 扩展器线缆导致磁盘延迟从 2ms 飙升至 8000ms，所有 IO 密集型服务全部瘫痪"
keywords: "磁盘IO延迟, iowait高, linux磁盘排查, iostat, iotop, linux IO瓶颈, 磁盘延迟飙升"
draft: false
featured: true
cover:
  image: ""
  caption: "磁盘 IO 延迟排查实录"
---

## 常见搜索关键词

如果您通过搜索引擎来到这篇文章，很可能正在经历以下症状之一：

- `top` 命令显示 `iowait` 高达 80-99%，服务器几乎无响应
- `iostat -x` 显示 `await` 高达数千毫秒
- PostgreSQL / MySQL 查询从正常的 10ms 变为 30 秒以上
- `ls`、`ssh` 等命令卡顿数十秒
- `dmesg` 被 `I/O error` 或 `SCSI` 相关消息刷屏
- 大量进程处于 `D` 状态（不可中断睡眠）
- 监控告警："节点磁盘延迟超过 5s"

本文完整还原了一起由 SAS 扩展器线缆故障导致磁盘延迟从 2ms 飙升到 8000ms 以上的真实生产事故，该事故导致 PostgreSQL 数据库集群和 5 个微服务全面瘫痪。

---

## 故障经过

### 环境信息

| 组件 | 规格 |
|---|---|
| 服务器 | Dell PowerEdge R750 |
| 存储 | JBOD 扩展柜，12 块 SATA SSD |
| 文件系统 | ext4 on LVM |
| 数据库 | PostgreSQL 15 |
| 应用服务 | 5 个微服务（Java/Go/Python） |
| 操作系统 | Ubuntu 22.04 LTS，内核 5.15 |

### 时间线

**14:23** — 应用监控面板全部飘红。P99 API 延迟从 50ms 飙升至 5 秒以上。PostgreSQL 查询时间从 10ms 变为 30s+。

**14:24** — 值班工程师收到告警：
- `node_disk_io_time_seconds_total` 显示持续高 IO 时间
- "节点磁盘延迟超过 5s" 阈值被突破
- PostgreSQL 连接池即将耗尽

**14:25** — SSH 登录节点成功，但每次按键回显需要 3-5 秒。命令执行需要 30-60 秒才能完成。

**14:26** — `top` 显示 `iowait` 高达 93%。负载均值 85（基线值：4-8）。

### 症状一览

- API 延迟：正常值的 100 倍
- `iowait`：90%+
- SSH：打字延迟严重，交互式会话几乎无法使用
- PostgreSQL：查询超时，连接池被占满
- 微服务：健康检查探测失败，实例被从服务发现中移除
- 系统日志：无法写入，`syslog-ng` 阻塞
- Prometheus：磁盘延迟指标远超 5s 阈值

---

## 背景知识

在深入排查之前，先快速回顾一下 Linux IO 栈以及 `iowait` 的真正含义。

### Linux IO 栈

当一个进程读写文件时，请求需要经过多个层次：

```
应用程序 (PostgreSQL)
    ↓
VFS（虚拟文件系统层）
    ↓
文件系统 (ext4)
    ↓
块设备层 (mq-deadline / kyber / none)
    ↓
SCSI 中间层（重试、超时处理）
    ↓
SCSI 底层驱动（HBA/控制器）
    ↓
物理设备 (SAS 扩展器 → SATA SSD)
```

每一层都增加了自己的排队、合并和超时逻辑。物理层的故障（例如 SAS 线缆故障）会逐层向上传播，表现为 IO 缓慢或失败。

### 什么是 iowait？

`top` / `mpstat` 中的 `iowait` 表示 CPU 在等待至少一个 IO 操作完成时处于空闲状态的时间百分比。

关键点：
- **iowait 是 CPU 空闲时间**，不是 CPU 忙碌时间。CPU 无事可做，因为它在等待 IO。
- 高 iowait 意味着**存储子系统是瓶颈**，而不是 CPU。
- iowait 不能告诉你哪个磁盘慢、哪个进程在做 IO、或者是读还是写。它只是一个信号，表明 IO 路径出了问题。

### IOPS vs 延迟

| 指标 | 测量内容 | 何时关注 |
|---|---|---|
| IOPS | 每秒 IO 操作数 | 吞吐量密集型工作负载 |
| 延迟 (await) | 每次 IO 操作从提交到完成的时间 | 延迟敏感型工作负载（数据库、实时系统） |
| %util | 设备忙于服务请求的时间百分比 | 在现代 NVMe/SSD 上不能表示饱和程度 |

在这次事故中，IOPS 下降是因为每个请求耗时 8000ms+，但关键指标是**延迟**，而不是 IOPS。

---

## 排查过程

### 第一步：检查系统负载

```bash
# 首先：查看 top 的输出
top
```

输出：
```
top - 14:26:18 up 42 days,  3:11,  2 users,  load average: 85.32, 42.18, 18.07
Tasks: 345 total,   1 running, 284 sleeping,  60 stopped,   0 zombie
%Cpu(s):  2.1 us,  1.8 sy,  0.0 ni, 93.7 id, 93.1 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem : 128898.6 total,   1245.3 free,  89234.5 used,  38418.8 buff/cache
MiB Swap:   2048.0 total,      0.0 free,   2048.0 used.  38894.2 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1234 postgres  20   0  ......
```

关键发现：
- `iowait` (wa)：**93.1%** — 严重
- 负载均值：**85.32** — 基线为 4-8，现在高了 10 倍
- 系统还活着，但几乎无法响应

```bash
uptime
# 14:26:18 up 42 days, 3:11, 2 users,  load average: 85.32, 42.18, 18.07
```

### 第二步：识别磁盘延迟

```bash
# 扩展磁盘统计，1 秒间隔，5 次采样
iostat -x 1 5
```

输出：
```
Device     r/s     w/s    rkB/s    wkB/s  await  svctm  %util
sda       12.3     8.7    456.2    234.1   2.5    0.8    1.7
sdb        0.4     0.2     12.3      4.1 8342.1 1230.5 75.3
sdc        8.2     5.1    321.4    123.7   3.1    0.9    1.2
```

关键发现：
- **sdb**：`await` 高达 **8342ms**（正常 SSD 为 2-5ms）
- **sdb**：`%util` 为 **75.3%** — 更重要的是延迟高得离谱
- 其他磁盘 (sda, sdc) 正常，说明这是**单盘问题**，而非控制器全域问题

`iostat -x 1 5` 命令说明：
- `-x`：扩展统计（包含 `await`、`svctm`、`%util`）
- `1 5`：每秒刷新一次，共 5 次

关键观察列：
- **await**：平均 IO 响应时间（含排队 + 服务时间）。这是最佳延迟指标。
- **svctm**：平均服务时间（实际 IO 处理时间，不含排队）。现代 SSD 应小于 1ms。
- **%util**：设备忙碌时间百分比。单个慢盘即使 IOPS 很低也能显示 100%。
- **r_await / w_await**：新版内核上读/写各自的 await。

{{< alert info >}}
在新版内核中，`svctm` 已被弃用，可能显示为 0。请改用 `await` + `r_await` / `w_await`。
{{< /alert >}}

### 第三步：识别 IO 密集型进程

```bash
# 只显示当前正在做 IO 的进程
iotop -o
```

输出：
```
TOTAL DISK READ: 0.4 K/s | TOTAL DISK WRITE: 0.2 K/s
  PID  PRIO  DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 1234 be/4    0.0 K/s    0.1 K/s  0.00 % 99.99 %  postgres: writer
 5678 be/4    0.0 K/s    0.0 K/s  0.00 % 99.99 %  syslog-ng
 9012 be/4    0.0 K/s    0.1 K/s  0.00 % 99.99 %  postgres: wal-writer
```

关键发现：
- **PostgreSQL writer 和 WAL writer**：卡在 99.99% IO 等待
- **syslog-ng**：也卡住了 — 系统连写日志都做不到
- 几乎没有实际吞吐量（0.4 K/s 读，0.2 K/s 写）— 磁盘实际上已经挂死

{{< alert warning >}}
当 syslog-ng 本身被 IO 阻塞时，你将无法记录事故过程。生产环境中建议使用独立日志磁盘或远程日志。
{{< /alert >}}

### 第四步：检查 SCSI 层

SCSI 中间层是理解磁盘 IO 问题的关键，因为它负责处理命令重试和超时。

```bash
# 检查设备状态
cat /sys/block/sdb/device/state
# → "running"

# 检查 SCSI 超时值（默认 30 秒）
cat /sys/block/sdb/device/timeout
# → 30
```

设备状态为 "running" — 内核仍然认为设备在正常工作。超时值为 30 秒，意味着 SCSI 层会等待 30 秒才宣布命令失败。

```bash
# 检查内核消息中的 SCSI 错误
dmesg | grep -i "scsi\|sdb\|I/O error" | tail -20
```

此场景预期输出：
```
[8372912.451] sd 0:0:1:0: task abort: scsi 0x00000000 failed to finish within 30s
[8372912.452] sd 0:0:1:0: attempting task abort! scsi(0x00000000)
[8372942.512] sd 0:0:1:0: task abort: scsi 0x00000001 failed to finish within 30s
[8372942.513] sd 0:0:1:0: attempting task abort! scsi(0x00000001)
[8372972.573] sd 0:0:1:0: task abort: scsi 0x00000002 failed to finish within 30s
[8372972.574] sd 0:0:1:0: attempting task abort! scsi(0x00000002)
[8373002.634] sd 0:0:1:0: task abort: scsi 0x00000003 failed to finish within 30s
```

每 30 秒，一个新的 SCSI 命令超时，中间层进行重试。这种模式确认问题出在 **SCSI/物理层**，而不是文件系统或应用程序。

```bash
# 也可检查 SCSI 错误计数器（并非所有内核都支持）
cat /sys/block/sdb/device/ioerr_cnt 2>/dev/null
# 可能不存在 — 取决于内核版本和驱动
```

### 第五步：检查磁盘 SMART 数据

```bash
# 受影响磁盘的 SMART 数据
smartctl -a /dev/sdb | grep -i "reallocated\|pending\|uncorrect\|error"
```

当磁盘本身健康但连接有故障时的预期输出：
```
SMART overall-health SELF-ASSESSMENT RESULT: PASSED
  5 Reallocated_Sector_Ct: 0
196 Reallocated_Event_Count: 0
197 Current_Pending_Sector: 0
198 Offline_Uncorrectable: 0
```

**SMART 数据看起来很干净** — 磁盘本身是健康的。这排除了 SSD 故障的可能性，明确指向**连接/控制器/扩展柜**问题。

{{< alert tip >}}
SMART 数据干净 + SCSI 超时 = 怀疑连接线路（线缆、扩展器、背板、HBA）。
SMART 数据恶化 + SCSI 超时 = 怀疑磁盘本身。
{{< /alert >}}

### 第六步：检查文件系统

```bash
# 检查文件系统级错误
dmesg | grep -i "ext4\|xfs\|journal" | tail -10
```

预期输出：
```
[8372912.455] EXT4-fs warning (device dm-0): ext4_end_bio:340: I/O error writing to inode 1234568
[8372972.576] EXT4-fs warning (device dm-0): ext4_end_bio:340: I/O error writing to inode 1234592
```

文件系统报告了 IO 错误，但这些是底层 SCSI 问题的**症状**，而不是根因。文件系统本身（ext4 日志、超级块）是完整的。

### 第七步：跟踪块设备层延迟

```bash
# 直接读取块设备统计信息
cat /sys/block/sdb/stat
```

输出格式（字段来自 Documentation/block/stat.txt）：
```
   read I/Os      merge  sectors   ticks     write I/Os     merge  sectors   ticks     in_flight  io_ticks  time_in_queue
   2847291        1245   88928345  23948231  8391234        2341   67123456  928371233  127        89234723  123781237
```

关键值是 `io_ticks` — 设备忙碌的总时间（毫秒）。如果这个值快速增长而 `in_flight` 不为零，说明设备卡住了。

在我们的案例中，sdb 的 `io_ticks` 每秒增长约 1000ms — 意味着设备 100% 忙碌，每次 IO 至少耗时 1 秒。

```bash
# 也可检查每盘延迟直方图（如果启用了 CONFIG_LATENCYTOP）
# 需要 root 权限，且依赖于发行版
# echo 1 > /proc/sys/kernel/latencytop  (如有)
```

---

## 根因分析

### 罪魁祸首：故障的 SAS 扩展器线缆

在将受影响磁盘下线并检查硬件后，发现 Dell R750 的 JBOD 扩展柜的 **SAS 扩展器线缆存在间歇性连接问题**。该线缆（可能在最近的机柜维护中）被部分松动，导致接触不良。

### 故障级联

以下是完整的事件链：

```
SAS 线缆接触不良
    ↓
SAS 扩展器与磁盘的链路间歇性断开
    ↓
发送到磁盘的 SCSI 命令超时（默认 30 秒）
    ↓
SCSI 中间层重试（默认最多 5 次）
    ↓
每次 IO 请求被阻塞 30-150 秒
    ↓
PostgreSQL WAL 刷盘挂起 → 所有写入阻塞
    ↓
PostgreSQL writer 进程进入 D 状态（不可中断睡眠）
    ↓
syslog-ng 尝试写入审计日志时阻塞
    ↓
所有 IO 密集型进程堆积在 D 状态
    ↓
系统负载飙升（负载均值 85+）
    ↓
节点几乎完全无响应
```

### 为什么每次 IO 需要 30-150 秒？

SCSI 中间层使用以下默认值：

| 参数 | 默认值 | 影响 |
|---|---|---|
| SCSI 超时 | 30 秒 | 声明命令失败前的等待时间 |
| 最大重试次数 | 5 | 放弃前的重试次数 |
| 最坏情况总计 | 30秒 × 5 = 150秒 | 一次 IO 请求可能被阻塞的最大时间 |

因此 PostgreSQL 的一次 `write()` 调用可能被阻塞长达 150 秒才会返回错误。在这 150 秒内，整个数据库实际上被冻结了。

### D 状态说明

等待 IO 的进程处于 **D 状态**（不可中断睡眠，TASK_UNINTERRUPTIBLE）。与 S 状态（可中断睡眠）不同，D 状态的进程**不能被杀死** — 即使是 `SIGKILL` 也不行。只有当 IO 操作完成或设备被下线时，它们才会退出 D 状态。

这就是为什么节点上有 60 多个 D 状态进程，以及为什么 `reboot` 被列为最后手段。

---

## 解决方案

### 紧急措施

{{< alert danger >}}
在执任何命令之前请仔细阅读。将包含活动数据的磁盘下线可能导致数据丢失。确保你已经识别出正确的磁盘。
{{< /alert >}}

#### 1. 识别受影响的磁盘

```bash
# 确认哪个磁盘有问题
smartctl -a /dev/sdb | grep -i "reallocated\|pending\|uncorrect"
dmesg | grep -i "sdb\|I/O error" | tail -10
```

#### 2. 检查文件系统布局

```bash
# 查找坏盘属于哪个 LVM VG/LV 和挂载点
lsblk /dev/sdb
# sdb           8:16   0   3.7T  0 disk
# └─vg_data-lv_data 253:0 0 3.7T 0 lvm  /var/lib/postgresql

# 如果是关键卷，检查是否有副本或能否故障转移
```

#### 3. 尝试 SCSI 总线重新扫描

有时内核可以通过总线重新扫描恢复设备：

```bash
# 重新扫描 SCSI 总线（host0 = 第一个 HBA，按需更改）
echo "- - -" > /sys/class/scsi_host/host0/scan
```

如果线缆问题是间歇性的，重新扫描可能重新建立连接。在我们的案例中没有成功 — 线缆已经坏到无法恢复。

#### 4. 将磁盘下线

如果重新扫描不起作用，将磁盘下线：

```bash
# 将磁盘下线（立即失败所有待处理的 IO）
echo offline > /sys/block/sdb/device/state
```

当你将设备设置为 `offline` 时，SCSI 层会立即失败所有待处理命令。这将导致：
- 所有 D 状态进程被释放（它们收到 IO 错误）
- PostgreSQL 提升副本或进入崩溃恢复
- 系统恢复响应

{{< alert danger >}}
将磁盘设置为 `offline` 会导致受影响文件系统上的文件系统错误。使用前必须重新挂载或执行 fsck。在我们的案例中，PostgreSQL 数据位于独立的 LV 上，可以故障转移到副本。
{{< /alert >}}

#### 5. 重新挂载文件系统（如需要）

如果文件系统损坏或处于不一致状态：

```bash
# 检查和重新挂载
umount /var/lib/postgresql
fsck -y /dev/vg_data/lv_data
mount /var/lib/postgresql
```

#### 6. 最后手段：重启

如果系统完全无响应且无法将磁盘下线：

```bash
# 同步并重启
sync; reboot
```

在极端情况下，可能需要硬重置（断电重启）。这应该是万不得已的手段。

### 长期修复措施

#### 1. 更换故障硬件

```bash
# 更换后，验证所有磁盘已被检测到
lsscsi
# [0:0:0:0]    disk    Dell     PERC H755        5.11  /dev/sda
# [0:0:1:0]    disk    ATA      INTEL SSDSC2KB   XCV1  /dev/sdb
# [0:0:2:0]    disk    ATA      INTEL SSDSC2KB   XCV1  /dev/sdc
```

#### 2. 减少 SCSI 超时以实现快速故障转移

```bash
# 将 SCSI 超时从 30 秒减少到 10 秒以加快检测速度
echo 10 > /sys/block/sdb/device/timeout

# 通过 udev 规则持久化
cat > /etc/udev/rules.d/99-scsi-timeout.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="scsi", ATTR{device/type}=="0", ATTR{device/timeout}="10"
EOF
```

#### 3. 为关键磁盘添加多路径 IO

```bash
# 安装多路径工具
apt-get install multipath-tools

# 配置多路径
cat >> /etc/multipath.conf << 'EOF'
defaults {
    user_friendly_names yes
    polling_interval 5
    path_selector "round-robin 0"
    path_grouping_policy multibus
}

multipaths {
    multipath {
        wwid  36xxxx...  # 替换为实际磁盘 WWID
        alias  mpath_data
    }
}
EOF

systemctl restart multipathd
```

#### 4. 建立监控体系

**Prometheus 告警规则（PromQL）：**

```yaml
# 磁盘延迟告警
- alert: DiskLatencyHigh
  expr: |
    rate(node_disk_io_time_seconds_total{device=~"sd[a-z]"}[5m])
    / rate(node_disk_reads_completed_total{device=~"sd[a-z]"}[5m])
    > 0.1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "磁盘 {{ $labels.device }} 平均延迟 > 100ms"

- alert: DiskLatencyCritical
  expr: |
    rate(node_disk_io_time_seconds_total{device=~"sd[a-z]"}[5m])
    / rate(node_disk_reads_completed_total{device=~"sd[a-z]"}[5m])
    > 1.0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "磁盘 {{ $labels.device }} 平均延迟 > 1s"

# SCSI 错误检测（需要 textfile collector 或自定义 exporter）
- alert: SCSIErrorsDetected
  expr: |
    node_scsi_errors_total > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "检测到 {{ $labels.device }} 上的 SCSI 错误"
```

**生产监控检查清单：**

```bash
# 使用 iostat 持续监控
iostat -x 60 > /var/log/iostat.log &

# 从 dmesg 监控 SCSI 错误
dmesg --follow | grep --line-buffered "I/O error" | while read line; do
    logger -p user.warning "SCSI_ERROR: $line"
done
```

---

## 经验教训

### 做得好的地方

1. **监控系统及时捕获了问题**：Prometheus 延迟告警在事故发生后 1 分钟内触发
2. **只读文件系统保护了我们**：部分文件系统在 IO 风暴下自动重新挂载为只读，防止了进一步损坏
3. **有副本可用**：PostgreSQL 副本在主库下线后接管了服务

### 做得不够的地方

1. **没有 SCSI 错误监控**：我们有磁盘延迟告警，但没有监控 SCSI 错误或 dmesg 模式
2. **存储路径单点故障**：没有多路径 IO。一根线缆故障不应该搞垮整个存储栈
3. **syslog-ng 位于同一磁盘**：日志守护进程与数据库争抢 IO，当磁盘故障时，我们丢失了事故期间的所有日志
4. **默认 SCSI 超时过长**：30 秒是为机械硬盘设计的。对于基于 SSD 的系统，10 秒或更短更合适
5. **没有 IO 延迟 SLO**：我们有 CPU 和内存 SLO，但没有磁盘延迟 SLO。本次事故后才添加了"节点磁盘延迟超过 5s"告警
6. **没有自动磁盘隔离机制**：将磁盘下线需要手动 SSH 介入 — 在 90% iowait 的情况下几乎不可能

### 关键要点

- **iowait 不做诊断，它只是发信号**。高 iowait 告诉你 IO 是瓶颈，但你仍然需要 `iostat`、`iotop` 和 `dmesg` 来找到原因。
- **SCSI 层是金丝雀**。SCSI 超时几乎总是表明硬件或线缆问题，而不是软件问题。
- **D 状态很危险**。处于不可中断睡眠的进程不能被杀死，会不断累积，使系统越来越糟。
- **监控 IO 路径，而不是只监控磁盘**。磁盘 SMART 数据可能显示"PASSED"，而连接到磁盘的线路已经断了。
- **做好线缆管理**。机柜维护时被部分松动的 SAS 线缆引发了整起事故。

---

## 总结

### 事故时间线

```
14:23 — 应用延迟飙升 100 倍（50ms → 5s+）
14:24 — 告警触发：节点磁盘延迟超过 5s
14:25 — 工程师 SSH 登录（极其缓慢）
14:26 — top 显示 93% iowait，负载均值 85
14:30 — iostat 定位到 sdb 的 await 高达 8000ms
14:35 — iotop 显示 postgres + syslog-ng 卡死
14:40 — dmesg 确认每 30 秒 SCSI 超时
14:45 — SMART 数据干净 — 怀疑线缆问题
14:50 — sdb 下线，进程释放
14:55 — PostgreSQL 副本接管，服务恢复
15:30 — 硬件检查确认 SAS 线缆松动
16:00 — 线缆重新插紧，磁盘验证，副本重新同步
总影响：约 90 分钟的部分或完全服务降级
```

### 故障流程图

```
                     ┌───────────────────────┐
                     │   SAS 线缆故障         │
                     │   (接触不良)            │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   SAS 扩展器链路        │
                     │   间歇性断开            │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   SCSI 命令超时         │
                     │   (默认 30 秒)          │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   SCSI 重试 (最多 5 次)  │
                     │   每次 IO 阻塞 30-150 秒 │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   块设备层停滞           │
                     │   mq-deadline 队列占满   │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   ext4 日志冻结          │
                     │   无法写入              │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   PostgreSQL WAL 阻塞    │
                     │   所有写入 D 状态        │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   系统无响应             │
                     │   93% iowait, 负载 85   │
                     └───────────────────────┘
```

### 命令速查表

| 命令 | 用途 |
|---|---|
| `top` | 快速检查：iowait、负载均值、D 状态进程 |
| `iostat -x 1 5` | 每盘延迟 (`await`)、利用率 (`%util`) |
| `iotop -o` | 当前在做 IO 的进程 |
| `dmesg \| grep -i scsi` | SCSI 错误和超时 |
| `cat /sys/block/sdX/device/state` | 磁盘设备状态 (running/offline) |
| `cat /sys/block/sdX/device/timeout` | SCSI 超时值（秒） |
| `smartctl -a /dev/sdX` | 磁盘健康状态 (SMART 数据) |
| `echo offline > /sys/block/sdX/device/state` | 紧急：将磁盘下线 |
| `echo "- - -" > /sys/class/scsi_host/host0/scan` | 重新扫描 SCSI 总线 |

---

*标签：storage, troubleshooting, linux, disk, performance*
*分类：Troubleshooting*
