---
title: "OOM Killer 误杀排查实录——当 Java 进程凭空消失"
date: 2026-05-25
weight: 100180
slug: "linux-oom-killer-troubleshooting"
tags: ["linux", "oom-killer", "java", "memory", "troubleshooting"]
categories: ["Linux"]
description: "Java 进程在高峰时段凭空消失，没有 crash 日志、没有堆 Dump——从怀疑代码 Bug 到追查到 OOM Killer 误杀，再到阻止内核再次越俎代庖的完整过程"
keywords: "oom killer, linux out of memory, java process killed, dmesg, overcommit, oom_adj"
draft: false
featured: true
cover:
  image: "/images/oom-killer-banner.svg"
  caption: "Linux OOM Killer 误杀排查"
---

# OOM Killer 误杀排查实录——当 Java 进程凭空消失

## 问题现象

周三下午 2 点，线上告警："支付对账服务进程不存在"。

不是"进程挂了"——是"不存在"。监控脚本发现 PID 没了，进程彻底消失了。

```bash
ps aux | grep payment-reconciliation
```

没有输出。进程真没了。

查应用日志：

```
14:02:15.123 [Thread-42] INFO  - Batch 20260525-001: processing record 5000/50000
14:02:15.456 [Thread-42] INFO  - Batch 20260525-001: processing record 5100/50000
# 然后日志就停了。没有错误，没有异常，没有警告
```

日志在 14:02:15 中断——之后什么都没有。

```bash
# JVM crash 日志（如果有的话会在工作目录下）
ls -la hs_err_pid*
```

没有 crash 日志。

```bash
# Java GC 日志
tail -20 gc.log
```

正常，没有 OOM 记录。堆内存还有余量。

环境：64GB 物理内存，Java 堆设置 -Xms32g -Xmx32g，部署在 CentOS 7 VM 上，未容器化。

**影响**：对账任务中断，当天账单无法生成。运维团队紧急重启后恢复，但第二天同一时间又挂了。连续三天，每天下午准时消失。

## 排查过程

### 错误尝试 1：怀疑 Java 代码 Bug

进程消失先怀疑代码。查了最近上线的发布——没有。查了日志最后几行——正在处理一批数据，看起来正常。查了数据库连接、线程池、队列深度——都正常。

确认不是代码问题。

### 错误尝试 2：查 JVM Crash 日志

Java 进程非正常退出一般会留下 hs_err_pid*.log 文件：

```bash
find / -name "hs_err_pid*" 2>/dev/null
```

没有。

JVM crash 的文件通常包含这些信息：

```
# A fatal error has been detected by the Java Runtime Environment:
#  SIGSEGV (0xb) at pc=0x...
# JRE version: ...
# Problematic frame:
# ...
```

没有这个文件，说明不是 JVM 内部崩溃。

```bash
# 检查是否被 systemd 或其他进程管理器杀了
journalctl -u payment-reconciliation.service | grep -i "killed\|signal\|main process"
```

没有记录。服务管理器认为进程正常退出（exit code 0），不是被重启的。

**踩坑点**：Java 进程被 OOM Killer 杀死时，**JVM 自己不知道发生了什么事**。OOM Killer 是内核直接发 SIGKILL，Java 进程连信号处理函数都跑不了。没有 hs_err、没有堆 Dump、没有退出日志——进程直接就没了。这是 OOM Killer 最坑的地方：它杀掉进程的方式跟你在终端敲 `kill -9` 一模一样，JVM 到死都不知道发生了什么。

### 错误尝试 3：怀疑运维操作

问了一圈——没人执行 kill 命令，没有部署操作，没有 systemd 重启。

```bash
# 审计日志
ausearch -k process_kill -ts 14:00 -te 14:10 2>/dev/null
```

如果配置了 auditd，可以看到谁杀了进程。但这次没有配置。

### 真正的发现：dmesg 里的 OOM Killer 记录

进程不明原因消失，除了被内核 OOM Killer 干掉，没有第二种可能。

```bash
dmesg -T | grep -i "killed process\|oom"
```

```
[Tue May 25 14:02:15 2026] java invoked oom-killer: gfp_mask=0x100cca(GFP_HIGHUSER_MOVABLE), order=0, oom_score_adj=0
[Tue May 25 14:02:15 2026] java: cpuset=/ mems_allowed=0
[Tue May 25 14:02:15 2026] CPU: 15 PID: 2345 Comm: java Killed
[Tue May 25 14:02:15 2026] Call Trace:
[Tue May 25 14:02:15 2026]  dump_header+0x4a/0x196
[Tue May 25 14:02:15 2026]  oom_kill_process+0xe6/0x120
[Tue May 25 14:02:15 2026]  out_of_memory+0x10c/0x260
[Tue May 25 14:02:15 2026]  __alloc_pages_slowpath+0x9e8/0xc60
[Tue May 25 14:02:15 2026]  __alloc_pages_nodemask+0x2b1/0x320
[Tue May 25 14:02:15 2026]  alloc_pages_vma+0x78/0x1e0
...
[Tue May 25 14:02:15 2026] Out of memory: Killed process 2345 (java) total-vm:48123456kB, anon-rss:42123456kB, file-rss:560kB, shmem-rss:0kB, UID:1000 pgtables:94520kB oom_score_adj:0
[Tue May 25 14:02:15 2026] oom_reaper: reaped process 2345 (java), now anon-rss:0kB
```

**实锤了**。OOM Killer 在 14:02:15 杀了 Java 进程，时间点跟日志中断的时间完全吻合。

再看一下系统当时的内存状况：

```bash
dmesg -T | grep -A 50 "oom-killer" | grep "Memory:" | head -5
```

```
[Tue May 25 14:02:15 2026] Memory: 58570164K/63872844K available (10234K kernel code, 1346K rwdata, 2980K rodata, 1956K init, 1888K bss, 5302680K reserved, 0K cma-reserved)
```

```
63872844K total - 58570164K available = 5.3GB 未被回收
```

```
[Tue May 25 14:02:15 2026] Node 0 active_anon:10531468kB inactive_anon:12kB active_file:4564kB inactive_file:23456kB
...
```

active_anon 10.5GB——这是已经分配出去的匿名内存页（堆 + 栈）。

```bash
# 看看被杀时的完整内存使用排行
dmesg -T | grep -A 30 "Killed process" | grep "\[ pid \]" -A 30
```

```
[ pid ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[ 1234]  1000  1234    123456     12345   123456        0             0 systemd-journal
[ 2345]  1000  2345  48123456  42123456   94520        0             0 java
[ 3456]  1000  3456   1234567    123456    12345        0             0 monitoring-agent
[ 4567]  1000  4567    123456     12345     1234        0             0 sshd
...
```

Java 的 RSS 41GB + 虚拟内存 48GB——其他进程加起来不到 2GB。

所以当时的内存状况是：
- 系统总物理内存：64GB（约 62GB 可用）
- Java RSS：41GB（堆 32GB + JVM 自身开销 + 线程栈 + Native 内存）
- 监控 agent：约 1GB
- Page Cache + Buffer：约 5GB
- 剩余空闲：约 5.3GB

等等——64GB 总内存，41GB 被 Java 用着，还剩 5.3GB，为什么会触发 OOM？

因为 **Linux 的 overcommit 机制**。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | Linux OOM Killer 判定系统内存不足，选择占用最大的 Java 进程杀掉 |
| 根本原因 | `vm.overcommit_memory=0`（默认），内核允许进程申请超过实际物理 + swap 的内存。Java 的 48GB 虚拟内存 + 其他进程 + 突发的 page cache 增长导致系统进入"commit limit 已满但内存还没用满"的状态，当有进程尝试申请新内存时触发 OOM |
| 触发条件 | 监控 agent 内存泄漏（约 1.2GB） + 日终 batch 处理带来大量 page cache 增长（约 4GB）+ Java RSS 本来就 41GB |
| 为什么每天同时段 | 监控 agent 在下午 2 点左右执行全量扫描，内存增长 + 日终 batch 的 page cache 需求同时达到峰值 |

**关键概念：Linux 内存 overcommit**

`overcommit_memory` 有三个值：

| 值 | 含义 | 表现 |
|----|------|------|
| 0（默认） | Heuristic overcommit | 不是严格检查，但也不是完全不管。当系统进入"接近内存耗尽"状态时，允许 overcommit 但也可能触发 OOM Killer |
| 1 | Always overcommit | 永远允许申请超过物理内存，类似于"你的虚拟内存地址空间不受物理限制" |
| 2 | Don't overcommit | 严格检查，不能超过 `overcommit_ratio` 的比例 |

`vm.overcommit_memory=0` 的情况下，当系统内存使用超过某个阈值（约 95-97%），内核认为"再这么下去要出事了"，主动触发 OOM Killer 来杀掉最"值"的进程——通常是占用内存最大的那个。

**Java 进程的虚拟内存为什么有 48GB？**

```
Java 堆：-Xmx32g → 32GB
元空间（Metaspace）：~1GB
线程栈（线程数 × 1MB）：~2GB
Code Cache：~256MB
Direct Buffer：~512MB
JVM 自身 Native：~1GB
---
总计虚拟内存 ~48GB，其中 ~41GB 是 RSS（实际物理内存）
```

核心教训：**Java 占的物理内存远比 -Xmx 大**。32GB 的堆配上各种 JVM 开销和线程栈，RSS 轻松到 40GB+。如果系统总内存只比 -Xmx 大一点点（比如 64GB 配 32GB 堆），遇上其他进程突发内存需求，就是 OOM Killer 的目标。

## 解决方案

### 紧急止血：重启进程 + 限制监控 agent

```bash
# 重启 Java 进程
systemctl start payment-reconciliation
```

```bash
# 同时限制监控 agent 的内存使用（临时）
systemctl stop memory-hungry-monitoring-agent
```

但这不是根本方案——第二天还会触发。

### 短期控制：调优 overcommit 策略

```bash
# 改为严格 overcommit 模式
echo 2 > /proc/sys/vm/overcommit_memory
```

持久化：

```
vm.overcommit_memory = 2
vm.overcommit_ratio = 90
```

```
echo "vm.overcommit_memory = 2" >> /etc/sysctl.conf
echo "vm.overcommit_ratio = 90" >> /etc/sysctl.conf
sysctl -p
```

**为什么 effective**：
- `overcommit_memory=2` 后，内核拒绝任何超过 `swap + RAM × overcommit_ratio` 的内存申请
- Java 启动时申请 48GB 虚拟内存，如果系统没有 64GB × 90% ≈ 57GB + swap 的可用额度，JVM 直接启动失败，而不是运行中被内核杀了
- 启动失败是"硬的"——你立刻知道内存不够，而不是在高峰时期才被莫名其妙地杀死

```bash
# 验证 overcommit 策略
cat /proc/sys/vm/overcommit_memory
# 2
```

### 根本修复：给 Java 留出足够的安全余量

```bash
# 1. 减小 Java 堆，留出系统余量
# -Xms24g -Xmx24g（从 32GB 降到 24GB，给系统其他进程留 8GB+）
```

```bash
# 2. 监控 agent 修复内存泄漏（具体修复略，这里是 Ops 视角）
```

```bash
# 3. 设置 oom_score_adj 保护关键进程
# 越低越不容易被杀，范围 -1000（保护）到 1000（优先杀）
echo -500 > /proc/2345/oom_score_adj
```

持久化需要通过 systemd service 或启动脚本：

```bash
# 在 systemd service 中设置
cat /etc/systemd/system/payment-reconciliation.service
```

```
[Service]
...
ExecStartPre=/bin/bash -c 'echo -500 > /proc/self/oom_score_adj'
ExecStart=/usr/bin/java -Xms24g -Xmx24g -jar app.jar
...
```

或者用 `cgroup` 保护：

```bash
# 创建 cgroup 并限制内存
mkdir /sys/fs/cgroup/memory/protected
echo 40G > /sys/fs/cgroup/memory/protected/memory.limit_in_bytes
echo 2345 > /sys/fs/cgroup/memory/protected/cgroup.procs
```

### 验证恢复

```bash
# 1. 确认 OOM Killer 没有再杀进程
dmesg -T | grep "Killed process" | grep java
# 没新的记录 = 安全

# 2. 确认内存使用在安全范围内
free -h
```

```
              total        used        free      shared  buff/cache   available
Mem:           62Gi        38Gi        12Gi        1.0Gi        12Gi        12Gi
```

```bash
# 3. 确认 overcommit 策略生效
cat /proc/sys/vm/overcommit_memory
# 2
```

```bash
# 4. 运行 batch 任务，观察峰值内存
# 通过一周的运行确认没问题
```

### 长期预防

```bash
# 1. 设置 proactive 告警——不要等 OOM 触发才报警
# 当 available memory < 10% 时提前告警
free -h | awk 'NR==2{print $7}'
# available 小于 5GB 时告警

# 2. 定期检查 dmesg 里的 OOM 记录
dmesg -T | grep -i "oom-killer\|killed process"
# 纳入监控系统巡检

# 3. 配置 vm.panic_on_oom=0（默认）
# 如果系统实在顶不住，不要 panic，让 OOM Killer 做事
# 反之如果希望内核挂了而不是乱杀进程：
vm.panic_on_oom = 1
kernel.panic = 10  # 10 秒后重启

# 4. 监控虚拟机/容器的 cgroup memory limit
# 如果使用 K8S，务必设置 Pod 的 memory request = limits
# 避免 Burstable QoS 导致 OOM 时被优先驱逐
```

## 教训总结

1. **Java 进程"凭空消失"——第一反应不是查日志，是查 dmesg。** 没有 hs_err、没有堆 Dump、没有退出码——这种"干净的消失"几乎一定是 OOM Killer。`dmesg -T | grep -i "killed process"` 这条命令应该是每个 Java 运维工程师的条件反射。

2. **-Xmx 32G ≠ 进程只吃 32G。** Java 的 RSS 比堆大得多——线程栈、元空间、Direct Buffer、Code Cache、JVM 自身 Native 内存，这些加起来能到堆的一半甚至更多。生产环境配内存的时候，堆建议不超过物理内存的 60-70%。

3. **overcommit_memory=0 是最危险的模式。** 它让进程在启动时感觉内存很充裕，但在运行期可能被内核"暗杀"。切换到 `overcommit_memory=2` 让内存不足在启动时就暴露出来——硬失败总比诡异的运行时 OOM 好十倍。

4. **OOM Killer 不一定是你的应用吃最多的内存才被杀。** 有时候是别的进程（监控 agent、sidecar）意外增长触发了 OOM，内核选了个"最大"的杀掉。`oom_score_adj` 可以保护关键进程，但最可靠的还是"确保系统有足够的内存余量"。把 Java 堆调小 8GB，给系统留出冗余，比任何内核参数调优都管用。
