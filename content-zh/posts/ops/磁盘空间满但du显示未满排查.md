---
title: "磁盘空间满但 du 显示未满排查——已删除文件仍占用空间的真相"
date: 2026-06-11
weight: 100520
slug: "disk-full-du-shows-no-data"
tags: ["linux", "storage", "troubleshooting", "disk", "filesystem"]
categories: ["Troubleshooting"]
description: "一个经典的 Linux 存储谜题——df 显示磁盘 100% 满，但 du 只显示使用了 30%。Java 应用打开了一个 70GB 的日志文件，logrotate 轮转删除了它，但 Java 从未关闭文件描述符，空间一直被内核保留"
keywords: "磁盘满 du显示未满, 已删除文件仍占用空间, lsof deleted files, linux磁盘空间排查, df和du不一致, 文件描述符泄漏"
draft: false
featured: true
cover:
  image: ""
  caption: "磁盘空间满但 du 显示未满排查"
---

# 磁盘空间满但 du 显示未满排查——已删除文件仍占用空间的真相

## 常见搜索词 / Common Search Queries

- 磁盘满了但 du 查不到大文件
- df 和 du 显示的磁盘使用量不一致
- 已删除的文件仍然占用磁盘空间
- lsof +L1 查到的 deleted 文件是什么
- 磁盘空间不足但 du 显示还有很多空间
- Linux 文件描述符泄漏导致磁盘空间异常
- logrotate 轮转后文件仍占用空间
- copytruncate 和 create 的区别

## 故障经过 / The Incident

### 环境信息

- **操作系统**: CentOS 7.9, kernel 3.10.x
- **应用**: Java 11 微服务（Spring Boot）
- **Web 服务器**: Nginx 1.16
- **日志系统**: syslog-ng, logrotate 每日轮转
- **监控**: Prometheus + Node Exporter + Alertmanager
- **分区**: `/var` 独立挂载，200GB 逻辑卷

### 时间线

14:23 — Prometheus Alertmanager 触发告警：生产节点 `/var` 分区磁盘使用率超过 90%，级别：严重。

14:25 — 运维人员登录服务器排查。

14:26 — `df -h /var` 确认：`/var` 使用率 **100%**。200GB 分区，可用块为 0。

14:28 — `du -sh /var/* | sort -rh | head -10` 只显示约 **60GB** 已识别数据。剩下的 140GB 去了哪里？

14:30 — 应用开始故障。Nginx 返回 **502 Bad Gateway**（无法写入访问日志）。Java 服务抛出 `java.io.IOException: No space left on device`。日志采集器停止工作。

14:32 — 宣布服务降级。启动应急响应。

### 症状汇总

| 症状 | 详情 |
|---|---|
| `df -h /var` | 100% 已满（200G / 200G） |
| `du -sh /var` | 约 60G（仅 30%） |
| inode 使用率 (`df -i`) | 正常，< 5% |
| 应用报错 | `No space left on device` |
| Nginx | 502 Bad Gateway |
| Prometheus | `node_filesystem_avail_bytes{...} 0` |

## 背景 / Background

要理解这个谜题，需要先回顾 Linux 文件管理的基础知识。

### 索引节点、目录项与文件描述符

Linux 中的每个文件有三层元数据：

1. **目录项（dentry）** — 目录列表中人类可读的文件名。`ls` 显示的就是它。它将文件名映射到 inode 编号。

2. **索引节点（inode）** — 存储文件除名称外的所有元信息：大小、权限、时间戳以及**数据块指针**（哪些磁盘块属于该文件）。每个 inode 有一个**链接计数（link count）**——指向该 inode 的目录项数量。

3. **数据块（Data Blocks）** — 存储在磁盘上的实际文件内容。

当进程打开一个文件时，内核在进程的 `/proc/<pid>/fd/` 命名空间中创建一个**文件描述符（FD）**。这个 FD 持有一个对 inode 的引用，与目录项无关。

### df 和 du 的区别

- **`du`**（disk usage）遍历**目录树**。它通过目录项找到 inode，然后累加每个 inode 引用的数据块。如果文件没有目录项（已被 unlink），`du` 看不到它。

- **`df`**（disk free）读取**文件系统超级块**，该超级块在块分配器层面跟踪总块数、已用块数和空闲块数。它统计所有已分配的块，不管它们是否有目录项。

这就是它们不一致的根本原因：`du` 看到的是目录树中链接的文件；`df` 看到的是磁盘上实际分配的所有块。

### 文件在打开状态下被删除会发生什么

当你执行 `rm` 删除文件时：

1. 内核删除目录项（从父目录中解除链接）。
2. inode 的链接计数减 1。
3. 如果链接计数变为 **0**，且没有进程持有该 inode 的打开 FD，则 inode 及其数据块被释放。
4. 但如果某个进程**仍然打开着这个文件**，链接计数变为 0 但 inode 仍然被分配，因为 FD 持有一个引用。数据块保留在磁盘上，`du` 看不到，但 `df` 会完整统计。

这正是我们遇到的情况。从目录的角度看文件已被"删除"，但它的数据块仍然健在，被一个打开的文件描述符挟持着。

## 排查过程 / Investigation

### 第一步：确认磁盘满

```bash
df -h /var
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/var 200G  200G     0 100% /var
```

200GB 全满，可用为 0。告警属实。

### 第二步：用 du 遍历目录树

```bash
du -sh /var/* | sort -rh | head -10
```

```
30G     /var/log
12G     /var/lib
8G      /var/cache
5G      /var/opt
3G      /var/spool
1G      /var/tmp
0.5G    /var/www
...
```

总计约 60GB。140GB 的差额——有东西在消耗空间，但在任何目录中都不可见。

### 第三步：检查 inode 使用率

inode 耗尽是一个已知的可能性，但很快排除了：

```bash
df -i /var
```

```
Filesystem       Inodes  IUsed   IFree IUse% Mounted on
/dev/mapper/var   13M    600K    12.4M    5% /var
```

inode 使用率仅 5%。问题一定是块分配层面的，不是 inode 耗尽。

### 第四步：查找被删除但仍打开的文件

经典工具是 `lsof` 配合 `+L1` 参数，它会列出链接计数为 0（已解除链接但仍打开）的文件：

```bash
lsof +L1 /var
```

```
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF   NLINK   NODE NAME
java     17234   root   12w   REG  253,0 72543272960   0  1835009 /var/log/java/app.log.1 (deleted)
```

找到了。一个 70GB+ 的文件，链接计数为 0（已删除），但 Java 进程仍然打开着。

或者，也可以通过 `/proc` 找到同样信息：

```bash
find /proc/*/fd -ilname "/var/*" 2>/dev/null | while read fd; do
  if [ ! -e "$fd" ]; then
    ls -la "$fd"
  fi
done
```

```
lrwx------ 1 root root 64 Jun 11 14:28 /proc/17234/fd/12 -> /var/log/java/app.log.1 (deleted)
```

### 第五步：检查 Java 进程的文件描述符

```bash
lsof -p 17234 | grep deleted | head -10
```

```
java  17234  root  12w  REG  253,0  72543272960  0  1835009  /var/log/java/app.log.1 (deleted)
```

文件描述符编号是 **12**，文件大小 **72,543,272,960 字节**（约 67.5 GiB / 72.5 GB）。

再看裸符号链接：

```bash
ls -la /proc/17234/fd/ | grep deleted
```

```
l-wx------ 1 root root 64 Jun 11 14:28 12 -> /var/log/java/app.log.1 (deleted)
```

### 第六步：计算已删除 FD 占用的总空间

```bash
lsof -p 17234 | grep deleted | awk '{print $7}' | paste -sd+ | bc
```

```
72543272960
lsof +L1 /var | sort -rn -k7 | head -5
```

确认约 **72.5 GB** 被已删除但未关闭的文件占用。这正好解释了差异：60GB（du 可见）+ 72.5GB（已删除 FD 持有）= 132.5GB，剩余部分是缓存/缓冲开销和其他小分配。

### 第七步：检查 logrotate 配置

```bash
cat /etc/logrotate.d/java-app
```

```
/var/log/java/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    postrotate
        /bin/kill -HUP `cat /var/run/java-app.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

logrotate 配置使用了默认的 **create** 策略：重命名旧日志，创建新空文件。postrotate 脚本尝试向 Java 进程发送 SIGHUP 以触发日志文件重新打开——但 Java 应用**没有处理 SIGHUP**，所以旧文件描述符一直保持打开。

### 第八步：检查内核文件句柄使用量

```bash
cat /proc/sys/fs/file-nr
```

```
9984   0   197583
```

已分配：9,984 | 未使用：0 | 最大：197,583。没有异常，不过排查过程还是要彻底。

## 根因 / Root Cause

根因链条非常清晰：

1. Java 微服务启动时打开 `/var/log/java/app.log` 进行写入。
2. 每天午夜 logrotate 将 `app.log` 重命名为 `app.log.1` 并创建新的 `app.log`。
3. logrotate 执行 postrotate 脚本，向 Java 进程发送 SIGHUP。
4. Java 进程**没有实现 SIGHUP 处理器**——信号被忽略，进程继续写入旧文件句柄，此时该句柄指向的是 `app.log.1`。
5. 下一次轮转时 logrotate 压缩 `app.log.1`（delaycompress 保留一个未压缩版本），但旧句柄仍然指向未压缩数据。
6. 随着时间的推移，`app.log.1` 增长到 72.5 GB。当 logrotate 下次运行时，它将 `app.log.1` 轮转为 `app.log.2.gz`，但 Java FD 仍然指向原始 `app.log.1` 的**已解除链接的 inode**，该 inode 现在已经完全脱离任何目录项。
7. 文件对 `du` 变得不可见，但内核保留了所有 72.5 GB 的数据块，因为 FD 仍然打开。

### 为什么 copytruncate 可以避免此问题

| 策略 | 机制 | 风险 |
|---|---|---|
| **create**（默认） | 重命名旧日志，创建新空文件 | 需要应用重新打开日志文件 |
| **copytruncate** | 复制文件内容到轮转文件，然后将原文件截断为 0 | 应用 FD 仍然指向同一个 inode——截断后空间立即释放 |

使用 `copytruncate`，文件永远不会被 unlink。inode 保持活跃，FD 保持有效，`truncate` 只是释放数据块。不需要应用配合。

## 解决方案 / Resolution

### 紧急处理——立即恢复磁盘

#### 方案 A：重启 Java 服务（推荐）

```bash
systemctl restart java-app
```

重启后：

```bash
df -h /var
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/var 200G  60G   140G  30% /var
```

磁盘立即降至 30%。当最后一个指向已 unlink inode 的文件描述符关闭时，72.5 GB 的数据块被释放。

#### 方案 B：发送 SIGHUP（如果应用支持）

```bash
kill -HUP 17234
```

在我们的案例中这没有效果，因为 Java 应用不处理 SIGHUP。请查阅你的应用文档。

#### 方案 C：直接截断文件描述符（零停机）

如果无法重启进程（例如不能接受停机），可以通过 `/proc` 直接截断文件：

```bash
truncate -s 0 /proc/17234/fd/12
```

```bash
df -h /var
```

这会立即释放空间，无需重启进程。但应用可能会行为异常——它发现文件从下面被截断（文件位置保持原始偏移，因此它会写入一个巨大的空洞然后从偏移 0 开始填充）。请谨慎使用，并在你的环境中先测试。

#### 验证恢复

```bash
df -h /var
lsof +L1 /var | wc -l   # 应该为 0
```

### 长期预防

#### 1. 在 logrotate 中使用 copytruncate

```bash
cat > /etc/logrotate.d/java-app << 'EOF'
/var/log/java/*.log {
    copytruncate
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
}
EOF
```

这是最可靠的修复方案。无需修改应用。`copytruncate` 将日志文件内容复制到归档文件，然后原地截断原始文件。应用的 FD 保持有效，截断后空间立即释放。

**注意**：`copytruncate` 有一个非常小的时间窗口，在复制和截断之间写入的日志数据可能会丢失。对大多数应用来说这是可接受的；如果要求零丢失，请看方案 2。

#### 2. 配置 Java 日志框架响应 SIGHUP

对于 Logback（Spring Boot 默认），可以添加一个 `ConfigurationEventListener` 在收到 SIGHUP 时重新打开文件，或者配置 `FixedWindowRollingPolicy` 配合外部轮转。Java 端应该与 logrotate 协作。

#### 3. 添加 FD 监控

监控残留的已删除文件描述符：

```bash
# Nagios / Icinga 检查
lsof +L1 /var | awk '{sum+=$7} END {print sum}'  # 已删除 FD 持有的总字节数

# 如果有任何已删除 FD 则告警
lsof +L1 /var > /dev/null 2>&1 && echo "WARNING: 发现已删除但未关闭的 FD"
```

Prometheus 规则思路（如果使用 node_exporter，需要自定义 textfile collector）：

```yaml
# /etc/prometheus/rules/disk.yml
groups:
  - name: disk
    rules:
      - alert: 已删除文件描述符占用空间
        expr: (node_filesystem_size_bytes{...} - node_filesystem_free_bytes{...}) - node_filesystem_avail_bytes{...} > 10 * 1024 * 1024 * 1024
        for: 10m
        annotations:
          summary: "df 与 du 的差异表明已删除文件仍在占用空间"
```

#### 4. 监控 df 与 du 的差异

定期运行一个简单检查：

```bash
#!/bin/bash
# /usr/local/bin/check-df-du-discrepancy.sh
MOUNT=$1
THRESHOLD_PCT=${2:-10}
DF_USED=$(df "$MOUNT" | tail -1 | awk '{print $3}')
DU_USED=$(du -s "$MOUNT" | awk '{print $1}')
DIFF=$(( (DF_USED - DU_USED) * 100 / DF_USED ))
if [ "$DIFF" -gt "$THRESHOLD_PCT" ]; then
  echo "CRITICAL: df ($DF_USED) 与 du ($DU_USED) 在 $MOUNT 上差异 ${DIFF}%"
  exit 2
fi
echo "OK: df 与 du 在 $MOUNT 上差异在 ${THRESHOLD_PCT}% 以内"
exit 0
```

#### 5. 更新应急手册

将 `df vs du` 差异的排查步骤文档化，确保任何值班工程师即使不了解这个边界情况也能按步骤操作。

## 经验教训 / Lessons Learned

1. **df 和 du 度量的是不同的东西。** 永远不要假设它们应该一致。`df` 报告块级别的分配；`du` 报告目录树的使用量。二者差异显著时，一定意味着有已分配但没有目录项的块存在。

2. **logrotate 的默认行为对长期运行的进程是危险的。** `create` 策略（重命名+新建文件）在应用不重新打开日志文件时悄悄失效。始终验证你的应用是否处理 SIGHUP，或者使用 `copytruncate`。

3. **信号处理很重要。** Java 应用在 JVM 启动时打开日志文件。JVM 对 SIGHUP 的默认行为是退出（Oracle JDK）或忽略（部分 OpenJDK 版本）。对于需要日志轮转的生产服务来说，两者都不正确。将 SIGHUP 处理测试纳入你的部署检查清单。

4. **监控正确的指标。** 仅 `df -h` 是不够的。应监控：
   - `df -h`（分区使用率）
   - `df -i`（inode 使用率）
   - `lsof +L1` 计数（已删除但打开的文件——煤矿中的金丝雀）
   - `df vs du` 差异（此类问题的早期预警）

5. **应急响应手册能节省大量时间。** 如果值班工程师手头有一份"磁盘满但 du 不一致"的排查文档，一小时的排查可以缩短到 10 分钟。

6. **内核信守承诺。** 当 Linux 说"数据还在那里"时，它是认真的——即使文件已被"删除"。内核会保持已分配的块直到最后一个文件描述符关闭。这是正确的行为，不是 bug，但可能会令人惊讶。

## 总结 / Summary

### 时间线

| 时间 | 事件 |
|---|---|
| 00:00 | logrotate 运行，将 `app.log` 轮转为 `app.log.1`，创建新的 `app.log` |
| 00:00 | Java 进程忽略 SIGHUP，继续写入旧 FD（现在指向 `app.log.1`） |
| 数天后 | `app.log.1` 增长到 72.5 GB；logrotate 后续夜间压缩但原始 inode 上的 FD 持续存在 |
| 14:23 | Prometheus 告警：`/var` 100% 满 |
| 14:25 | 运维人员登录 |
| 14:26 | `df -h` 确认 100% |
| 14:28 | `du -sh /var/*` 只显示 60GB——发现谜题 |
| 14:32 | `lsof +L1 /var` 发现 72.5 GB 已删除 FD |
| 14:35 | `systemctl restart java-app`——磁盘降至 30% |
| 14:36 | 服务恢复。事件解决。 |

### 命令对比表

| 命令 | 显示内容 | 何时使用 |
|---|---|---|
| `df -h` | 块级别分配（总块数、已用块数） | 磁盘满时的第一步检查 |
| `df -i` | inode 分配计数 | 当 df -h 显示有空间但应用报"no space"时 |
| `du -sh <路径>` | 目录树聚合用量 | 查找哪些目录占用空间 |
| `lsof +L1 <挂载点>` | 链接计数为 0 的文件（已删除但仍打开） | df 和 du 显著不一致时 |
| `find /proc/*/fd -ilname` | 通过 /proc 实现同 lsof +L1 | 当 lsof 未安装时 |
| `ls -la /proc/<pid>/fd/` | 列出进程所有打开的 FD | 检查特定进程 |
| `truncate -s 0 /proc/<pid>/fd/<n>` | （操作）将 FD 截断为 0 | 无需重启的紧急空间恢复 |
| `cat /proc/sys/fs/file-nr` | 内核 FD 分配统计 | 怀疑 FD 耗尽时 |

### 核心要点

当 `df` 显示磁盘已满但 `du` 不认同时，你几乎一定是在面对一个持有已删除文件打开文件描述符的进程。修复方法是使用 `lsof +L1` 识别进程，然后重启该进程或截断 FD。长期预防措施是在 logrotate 中配置 `copytruncate`，或确保你的应用正确处理日志轮转信号。
