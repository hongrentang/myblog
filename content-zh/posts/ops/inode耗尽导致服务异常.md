---
title: "inode 耗尽导致服务异常——明明有空间却写不进文件"
date: 2026-05-20
weight: 100070
slug: "linux-inode-exhaustion-troubleshooting"
tags: ["linux", "filesystem", "troubleshooting"]
categories: ["Linux"]
description: "完整复盘一次 inode 耗尽故障——从误判磁盘空间不足到定位 /var/spool/postfix 百万小文件"
keywords: "inode耗尽, df -i, 磁盘空间不足, no space left on device, postfix maildrop, linux排障"
draft: false
featured: true
cover:
  image: "/images/linux-inode-exhaustion-banner.svg"
  caption: "Linux inode 耗尽排查"
---

# inode 耗尽导致服务异常——明明有空间却写不进文件

## 背景

周二早上刚到工位，告警群就炸了。一组线上服务器的日志采集突然停了，监控显示 "No space left on device"。

我下意识 `df -h` 看了一下：

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G   12G   38G  24% /
```

24%？哪来的空间不足？

但应用确实在报错。Java 进程疯狂写 `IOException: No space left on device`，日志采集器也挂了。再拖下去，磁盘写不了数据，业务数据可能会丢。

## 排查过程

### 错误尝试 1：怀疑是 df 不准，检查了文件系统

第一反应是 `df` 缓存没刷新。我试了用 `sync` 强制刷写，又用 `fdisk -l` 确认分区大小，都没问题。

```bash
sync
df -h /
```

结果一样，24%。

这时候我做了最蠢的一件事——**直接在生产环境跑了一次 fsck**。

```bash
# 别学我，生产环境别这么干
fsck -n /dev/vda1
```

好在加了 `-n`（只检查不修复），跑完啥也没发现。但是这个过程阻塞了磁盘 IO，导致线上服务多了几秒延迟——**毫无意义的自选操作**。事后复盘，这一步浪费了 15 分钟，还引入了不必要的风险。

**踩坑点**：`df -h` 显示有空间，但应用报 "No space"——第一反应不应该是怀疑文件系统损坏，而是应该查 **inode**。

### 错误尝试 2：怀疑是进程文件句柄耗尽

"会不会是进程打开了太多文件，把文件句柄上限打满了？" 我当时这么想。

```bash
# 查当前进程打开的文件数
lsof | wc -l
```

```
83521
```

八万多个？好像不少。查一下系统限制：

```bash
ulimit -n
```

```
1024
```

等等，8 万 > 1024？这不科学。后来才想起来 `ulimit -n` 是 **per-process** 的限制，不是系统全局的。`lsof | wc -l` 统计的是所有进程的总和。

```bash
# 真正应该查的是每个进程的 fd 数
for pid in /proc/[0-9]*; do
    fd_count=$(ls "$pid/fd" 2>/dev/null | wc -l)
    if [ "$fd_count" -gt 1000 ]; then
        echo "$(cat $pid/comm 2>/dev/null) (PID $(basename $pid)): $fd_count fds"
    fi
done | sort -t: -k2 -rn | head -5
```

```
java (PID 12345): 2341 fds
dockerd (PID 987): 1567 fds
```

Java 进程开了 2300+ 个文件描述符，但离系统上限（`cat /proc/sys/fs/file-max` 通常是百万级）还差得远。文件句柄不是问题。

### 错误尝试 3：怀疑是磁盘 IO 问题或硬件故障

"会不会是磁盘有坏道导致 IO 错误？" 我看了 smartctl 和 iostat：

```bash
smartctl -a /dev/vda | grep -E "Reallocated|Pending|Offline"
```

```
  5 Reallocated_Sector_Ct   100   100   010    -    0
197 Current_Pending_Sector   100   100   000    -    0
```

没有坏道。

```bash
iostat -x 1 3
```

```
Device   r/s   w/s   await  %util
vda      2.3   1.8   1.2    0.8
```

IO 延迟 1.2ms，util 不到 1%——磁盘闲得很。

这是纯软件层面的报错，不是硬件问题。但我这时候已经绕了快 30 分钟了。

### 真正的发现

同事路过扫了一眼我的屏幕，说了一句："你查过 inode 吗？"

我当时甚至愣了一下——inode 是什么来着？

```bash
df -i /
```

```
Filesystem     Inodes  IUsed   IFree IUse% Mounted on
/dev/vda1      3.2M    3.2M      87  100% /
```

**100%**。inode 用完了。文件系统可以分配的数据块还剩很多（38G），但 inode 号全部耗尽了，任何创建新文件的操作都会返回 "No space left on device"。

这就是为什么 `df -h` 显示 24% 但应用写不了文件——**硬盘有两个维度：数据块和 inode，任何一个满了都无法写入**。

找到方向了，接下来就是定位谁吃了 inode。

```bash
# 查哪个目录文件数量最多
for dir in /*; do
    count=$(find "$dir" -xdev 2>/dev/null | wc -l)
    echo "$count $dir"
done | sort -rn | head -5
```

```
1425621 /var
...
```

`/var` 下有 140 多万个文件。钻进去：

```bash
for dir in /var/*; do
    count=$(find "$dir" -xdev 2>/dev/null | wc -l)
    echo "$count $dir"
done | sort -rn | head -5
```

```
1347289 /var/spool
...
```

`/var/spool` 下 130 多万个。继续：

```bash
ls -la /var/spool/postfix/maildrop/ | wc -l
```

```
1123457
```

113 万个小文件在 `postfix maildrop` 目录里。每封邮件（因为 crontab 输出未重定向而被 postfix 捕获）就是一个几字节的小文件，每个文件消耗一个 inode。

```bash
# 确认一下多小
ls -la /var/spool/postfix/maildrop/ | head -5
```

```
-rw------- 1 postfix postfix 328 May 20 03:47 B1C2D3E4F5
-rw------- 1 postfix postfix 297 May 20 03:48 G6H7I8J9K0
-rw------- 1 postfix postfix 341 May 20 03:48 L1M2N3O4P5
```

每封邮件 300 多字节，113 万封加起来才 350MB 左右——对数据块来说微不足道，但吃掉了 110 万个 inode。整个根分区的 inode 总数只有 320 万。

## 根因分析

- **直接原因**：`/var/spool/postfix/maildrop/` 积累了 110 万+ 个小文件，耗尽 inode
- **根本原因**：系统上有大量 crontab 任务没有重定向输出。cron 默认会把任务输出通过邮件发送给用户，而 postfix 是系统默认的 MTA。结果就是每个没有重定向的 cron 输出都变成一封小邮件堆积在 maildrop
- **辅助因素**：postfix 没有配置自动清理，系统也没有 inode 监控

一条没有重定向的 crontab 长这样：

```bash
# 错误的写法——cron 会把输出通过 sendmail 发邮件
*/5 * * * * /usr/local/bin/healthcheck.sh

# 正确的写法——重定向到 null 或日志文件
*/5 * * * * /usr/local/bin/healthcheck.sh > /dev/null 2>&1
```

我们系统上有几十个这样的定时任务，跑了几年没人管。积少成多，百万封邮件就是这个来的。

## 解决方案

### 紧急处理：释放 inode

找到问题后先止血。删 maildrop 下的小文件：

```bash
# 不能用 rm -rf，文件太多会报参数列表过长
# 正确做法：find + delete
find /var/spool/postfix/maildrop/ -type f -delete
```

这个命令跑了大概 8 分钟——110 万个文件，删除本身也需要 IO。

或者用 `xargs` 分批删：

```bash
# 效果同上，适合没有 -delete 的老系统
ls /var/spool/postfix/maildrop/ | xargs -P 4 -I {} rm -f /var/spool/postfix/maildrop/{}
```

清理过程中可以用 `watch` 观察 inode 释放：

```bash
watch -n 5 'df -i /'
```

```
Every 5.0s: df -i /

Filesystem     Inodes  IUsed   IFree IUse% Mounted on
/dev/vda1      3.2M    2.1M   1.1M   66%  /
```

删完确认服务恢复：

```bash
# 确认磁盘恢复正常
df -i /
df -h /

# 重启应用验证写入
systemctl restart log-collector
journalctl -u log-collector --since "10 minutes ago" | grep -i error
```

✅ **恢复验证**：
- `df -i` 显示 IUse% 降到正常水平（30% 以下）
- 应用不再报 "No space left on device"
- 日志采集器恢复正常推送

### 长期修复：防止复发

```bash
# 1. 关闭 postfix，这台服务器不需要发邮件
systemctl stop postfix
systemctl disable postfix

# 2. 或者保留 postfix 但清空 maildrop（如果确实需要邮件功能）
# 添加定时清理任务
cat > /etc/cron.daily/clean-maildrop << 'EOF'
#!/bin/bash
find /var/spool/postfix/maildrop/ -type f -mtime +7 -delete
EOF
chmod +x /etc/cron.daily/clean-maildrop

# 3. 修改所有 crontab，重定向输出
# 查哪些 crontab 没有重定向
grep -r "^[^#].*[a-zA-Z0-9]" /var/spool/cron/crontabs/ | grep -v ">/dev/null" | grep -v ">&/dev/null"
```

修改后的 crontab 写法：

```bash
# 正确示例——所有输出都重定向
*/5 * * * * /usr/local/bin/healthcheck.sh > /dev/null 2>&1
0 3 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

### 预防：添加 inode 监控

```bash
# Prometheus 告警规则
# node_filesystem_files_free / node_filesystem_files < 0.1
# 即 inode 使用率超过 90% 时告警
```

对比一下以前没有监控时的后果：

| 指标 | 之前 | 之后 |
|------|------|------|
| 磁盘监控 | 只监控块使用率（df -h） | 块 + inode 双维度监控 |
| cron 规范 | 无要求 | 所有 cron 必须重定向输出 |
| postfix 管理 | 默认开启 | 不需要发邮件的服务器直接禁用 |
| 小文件清理 | 无 | 每月自动清理 7 天前的 maildrop |

## 教训总结

1. **"No space left on device" 不一定是磁盘空间满了**。`df -h` 只是第一层，`df -i` 才是很多人忽略的第二层。文件系统有两个维度，任何一个满了都会报同样的错误。

2. **crontab 的默认行为是个坑**。没有重定向输出的 cron 任务会把所有输出通过邮件发送，积少成多就是百万个小文件。几十个 cron 任务跑几年，后果就和你看到的这张 inode 100% 截图一样。

3. **生产环境别一上来就跑 fsck**。我那次 fsck 虽然没有破坏数据，但阻塞了磁盘 IO 造成了几秒延迟。排查应该从最轻量、最安全的命令开始：先用 `df -i`，再用 `find` 定位，最后才考虑底层诊断。

4. **inode 和小文件是共生关系**。大量小文件（容器日志、临时缓存、session 文件、邮件队列）的共同特征：占用极少的磁盘空间，但消耗大量的 inode。监控系统建议同时检查块使用率和 inode 使用率。

5. **postfix 在不需要发邮件的服务器上应该被禁用**。很多 Linux 发行版默认安装并启动 postfix，但大部分服务器根本不需要 MTA 功能。一个 `systemctl disable postfix` 就能避免 inode 被 maildrop 耗尽。
