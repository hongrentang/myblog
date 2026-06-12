---
title: "Linux 进程 D 状态深度排查——当所有进程冻结，没有任何信号能唤醒它们"
date: 2026-06-12
weight: 100540
slug: "linux-process-d-state-investigation"
tags: ["linux", "kernel", "troubleshooting", "process", "system"]
categories: ["Troubleshooting"]
description: "Linux 进程 D 状态（不可中断睡眠）深度排查实录——NVMe 驱动 bug 导致数十个进程陷入 D 状态，生产服务器半死不活，最终紧急重启恢复"
keywords: "linux D状态, 不可中断睡眠, 进程卡死, D状态排查, 内核 hung task, nvme 驱动挂死, 进程冻结"
draft: false
featured: true
cover:
  image: ""
  caption: "Linux 进程 D 状态深度排查"
---

# Linux 进程 D 状态深度排查——当所有进程冻结，没有任何信号能唤醒它们

## 常见搜索关键词

如果你是从搜索引擎来到这里的，你可能正在搜索以下内容：

- linux 进程 D 状态 不可中断睡眠
- D 状态进程如何排查
- kill -9 杀不掉 D 状态进程
- 负载高全是 D 状态进程
- hung_task_timeout_secs blocked for more than 120 seconds
- nvme 驱动挂死 D 状态恢复
- 内核 hung task 看门狗触发
- 如何查看 D 状态进程在等什么
- D 状态和 Z 状态的区别
- Linux 紧急重启 D 状态进程

---

## 故障经过

**环境信息：**

| 组件 | 版本 |
|------|------|
| 操作系统 | Ubuntu 22.04 LTS |
| 内核版本 | 5.15.0-86-generic |
| 存储设备 | NVMe SSD (Intel P5510) |
| 文件系统 | ext4 |
| 数据库 | PostgreSQL 15 |
| 缓存 | Redis 7 |
| 应用运行时 | Node.js 18 |

**时间线：**

一个平静的周二下午。PostgreSQL 的 pg_cron 作业按计划触发了每小时一次的 VACUUM 操作——一切看起来都很正常。然后，在 14:37，监控告警突然爆发。

**症状：**

- SSH 可以连接，但每条命令都需要 30 秒以上才能执行完成
- `ls /` 需要 30 秒才能返回结果
- `ps aux` 显示 30+ 个进程处于 `D` 状态
- 服务器可以 ping 通，但 TCP 连接（HTTP、PostgreSQL 客户端连接）全部超时
- 32 核机器的负载平均值超过 100
- `dmesg` 中出现内核 hung task 消息

这台服务器从技术上讲还"活着"，但实际上已经死亡。没有新的 PostgreSQL 连接能够建立。Node.js 应用不断向上游返回 502 错误。Redis 还能响应本地命令，但无法持久化到磁盘。

---

## 背景

### Linux 进程状态

每个 Linux 进程都处于以下几种状态之一：

| 状态 | 名称 | 说明 |
|------|------|------|
| R | 运行/可运行 | 进程正在运行或等待 CPU 时间片 |
| S | 可中断睡眠 | 进程正在等待某个事件（如网络数据），可以被信号中断 |
| D | 不可中断睡眠 | 进程正在等待 I/O 完成，不能被任何信号中断 |
| T | 停止 | 进程已被停止（SIGSTOP/SIGTSTP） |
| Z | 僵尸 | 进程已终止，但父进程尚未读取其退出码 |

### D 状态的含义

D 状态代表"不可中断睡眠"（Uninterruptible Sleep）。当一个进程处于 D 状态时，它被阻塞等待某个 I/O 操作完成。内核让这个进程进入睡眠状态，并且**没有任何信号可以唤醒它**——SIGTERM 不行，SIGKILL（信号 9）也不行。只有当 I/O 操作完成（成功或失败）后，进程才会离开 D 状态。

这个机制的存在是因为内核必须保证某些 I/O 操作原子性地完成。如果允许进程在持有内核锁或存储写入正在进行时被杀死，内核可能陷入不一致的状态——文件系统元数据损坏、数据丢失，甚至内核崩溃。

### D 状态与 Z 状态的区别

这两个状态经常被混淆：

- **D 状态（不可中断睡眠）：** 进程仍然活着。它被阻塞在 I/O 上。它不是僵尸。进程没有退出——它无法退出，因为内核在 I/O 完成之前不允许它退出。
- **Z 状态（僵尸）：** 进程已经退出（终止）。只有它的进程描述符还留在进程表中，等待父进程调用 `waitpid()`。

D 状态进程会消耗 CPU 资源（它会参与负载平均值的计算）。Z 状态进程不消耗 CPU，但会占用一个 PID 表条目。

### 内核 Hung Task 检测机制

Linux 内核有一个内置的看门狗机制，用于检测卡在 D 状态的进程。`hung_task` 内核线程定期运行（每 `hung_task_check_interval_secs`，默认 120 秒），检查是否有任何任务处于 D 状态的时间超过 `kernel.hung_task_timeout_secs`（默认 120 秒）。

当检测到卡住的任务时，会记录类似这样的消息：

```
INFO: task postgres:1234 blocked for more than 120 seconds.
      Tainted: P           O      5.15.0-86-generic
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
```

默认情况下，hung task 检测器只**记录**警告——它不会采取任何恢复措施。但某些发行版会将内核配置为触发 panic 或 crash dump（`hung_task_panic = 1`）。

---

## 排查过程

当服务器部分挂死但 SSH 仍然可用时，你有一个狭窄的时间窗口来收集诊断数据。你运行的每条命令都在与已经阻塞的 I/O 资源竞争。所以要精准、高效。

### 第一步：评估损害范围

首先，确认问题的范围：

```bash
uptime
```

```
 14:38:12 up 187 days,  3:42,  1 user,  load average: 124.37, 89.52, 42.18
```

32 核机器负载平均值 124。显然出了大问题。

```bash
top
```

在 `top` 中，查看进程状态摘要行。如果显示数十个进程处于 `D` 状态，说明发生了系统性的 I/O 挂死。

### 第二步：识别 D 状态进程

列出所有当前处于 D 状态的进程：

```bash
ps -eo pid,stat,wchan,comm | grep "^ *[0-9] D"
```

示例输出：

```
 2341 D     ?          postgres: checkpointer
 2392 D     ?          postgres: walwriter
 2456 D     ?          postgres: autovacuum worker
 2678 D     ?          redis-server
 3123 D     ?          node app-server
 3190 D     ?          node worker
 ...
```

`wchan` 列显示进程阻塞在内核的哪个函数中。如果它们都显示相同或相似的函数（例如 `nvme_wait_ready`、`blk_mq_get_tag`、`io_schedule`），说明挂死是系统性的，很可能发生在块设备或驱动层。

另一种方法：

```bash
ps aux | awk '$8 ~ /D/ {print}'
```

### 第三步：检查内核 Hung Task 消息

```bash
dmesg | grep -i "hung_task\|blocked for more than\|D state" | tail -20
```

寻找类似这样的模式：

```
[1034567.890] INFO: task postgres:2456 blocked for more than 120 seconds.
[1034567.890] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[1034567.890] postgres       D    0  2456   2345 0x00000000
[1034567.890] Call Trace:
[1034567.890]  __schedule+0x2d6/0x960
[1034567.890]  schedule+0x4b/0xc0
[1034567.890]  io_schedule+0x42/0x70
[1034567.890]  wait_on_page_bit+0xe0/0x130
[1034567.890]  __filemap_fdatawait_range+0x121/0x190
[1034567.890]  filemap_fdatawait+0x25/0x30
[1034567.891]  ext4_sync_file+0x8d/0x1c0
[1034567.891]  do_fsync+0x38/0x60
[1034567.891]  __x64_sys_fsync+0x10/0x20
[1034567.891]  do_syscall_64+0x43/0x90
[1034567.891]  entry_SYSCALL_64_after_hwframe+0x62/0xcc
[1034567.891] 
[1034567.892] INFO: task redis-server:2678 blocked for more than 120 seconds.
[1034567.892] ... (similar call trace)
```

这里的关键观察是：**所有**被阻塞的任务都卡在 `io_schedule` 或 `wait_on_page_bit` 中。这告诉你挂死发生在 I/O 层，而不是应用代码中。它们都在等待块层完成读或写操作。

### 第四步：查看每个 D 状态进程在等待什么

要精确查看某个进程阻塞在哪个内核函数上：

```bash
cat /proc/<pid>/stack
```

示例：

```
[<0>] io_schedule+0x42/0x70
[<0>] wait_on_page_bit+0xe0/0x130
[<0>] __filemap_fdatawait_range+0x121/0x190
[<0>] filemap_fdatawait+0x25/0x30
[<0>] ext4_sync_file+0x8d/0x1c0
[<0>] do_fsync+0x38/0x60
```

这确认了进程正在等待一个页面写入磁盘（ext4 文件系统层），而它又在等待块层完成写入。

也可以使用：

```bash
cat /proc/<pid>/wchan
```

这会给出原始的内核函数名：

```
io_schedule
```

如果所有 D 状态进程都显示 `io_schedule`、`blk_mq_get_tag` 或 `nvme_wait_ready`，问题出在 NVMe 驱动或块层。

### 第五步：检查 I/O 统计

```bash
iostat -x 1
```

当系统挂死时，`iostat` 本身可能也会挂住，或者显示极高的 `await` 和 `svctm` 值。如果 `iostat` 挂住了，跳过实时工具，直接读取原始块设备统计：

```bash
cat /sys/block/nvme0n1/stat
```

stat 文件包含 11 个字段。重点关注：

- 字段 3 (rd_ticks)：读取操作花费的总毫秒数
- 字段 7 (wr_ticks)：写入操作花费的总毫秒数
- 字段 10 (io_ticks)：设备有 I/O 请求排队的总毫秒数

如果 `io_ticks` 远大于运行时间，说明设备一直持续繁忙——或者卡死了。

### 第六步：检查 NVMe 驱动状态

```bash
cat /sys/class/nvme/nvme0/device/state
```

预期输出：

```
live
```

如果显示其他值（如 `dead`、`error`），说明驱动检测到了错误状态并进入了恢复流程。

```bash
cat /sys/class/nvme/nvme0/device/controller/cntlid
```

检查 NVMe 错误日志：

```bash
nvme list
```

```bash
nvme error-log /dev/nvme0n1 | head -20
```

错误日志可能显示：

```
Error Log Entries for device:nvme0 entries:64
 .................
 Entry[ 0]   
 .................
 error_count     : 3
 sqid            : 0
 cmdid           : 0x003e
 status_field    : 0x4004 (INVALID SQ ID)
 phase_tag       : 0
 parm_err_loc    : 0x0000
 lba             : 0x00000000000000ff
 nsid            : 0x00000001
 vs              : 0
```

`INVALID SQ ID` 错误是一个危险信号——它表明驱动试图将命令提交到一个不存在或已停用的提交队列。

### 第七步：检查内核日志中的 NVMe 特定错误

```bash
dmesg | grep -i "nvme\|pci\|msi\|irq" | tail -20
```

寻找类似这样的消息：

```
nvme nvme0: I/O 214 QID 2 timeout, aborting
nvme nvme0: I/O 215 QID 2 timeout, aborting
nvme nvme0: I/O 216 QID 2 timeout, aborting
nvme nvme0: Abort status: 0x1000
nvme nvme0: Unable to abort command 214
nvme nvme0: controller is down; will reset: LST=0 MST=0
nvme nvme0: Device not ready; aborting reset, CSTS=0x0
...
nvme nvme0: Removing after probe failure status: -19
```

"Unable to abort command"后跟"controller is down"和"Device not ready"的模式，是驱动程序级别挂死的典型特征——NVMe 控制器完全停止响应中断。

### 第八步：检查中断配置

```bash
cat /proc/interrupts | grep nvme
```

```
  CPU0   CPU1   CPU2   ...  CPU31
 12345      0      0    ...      0  nvme0q0, nvme0q1
     0  23456      0    ...      0  nvme0q2
     0      0  18293    ...      0  nvme0q3
  ...
```

如果一个或多个队列显示零中断，而其他队列计数正常，说明该队列的中断已经丢失。这就是决定性证据。

```bash
cat /sys/class/nvme/nvme0/device/irq
```

这会显示分配给 NVMe 设备的 IRQ 号。与 `/proc/interrupts` 交叉对比，查看哪些向量是活跃的。

---

## 根因

在收集了以上所有数据后，根因变得清晰。以下是事件的完整链条：

### 1. PostgreSQL VACUUM 触发大量 I/O

PostgreSQL 的自动清理工作进程对一个大型表（约 120 GB）发起了 VACUUM 操作。VACUUM 进程执行顺序扫描并对脏页发起回写。PostgreSQL 使用 `fsync()` 来确保数据完整性，这会强制内核同步地将页面写入磁盘。

在这种特定工作负载下，VACUUM 同时发起了超过 256 个 I/O 请求，完全占满了 NVMe 队列深度。

### 2. MSI-X 中断向量耗尽

Intel P5510 NVMe SSD 最多支持 128 个 MSI-X 中断向量。NVMe 驱动为每个 I/O 队列分配一个中断向量。当工作负载在所有可用队列上同时发起 I/O 请求时，NVMe 控制器上的 MSI-X 向量表达到极限。

NVMe 规范允许每个队列最多 64K 个命令的队列深度，但中断向量是一种有限的硬件资源。当所有向量都在使用中，而需要产生新的中断时，控制器必须重用现有向量——如果时机不对，完成中断可能丢失。

### 3. 完成中断丢失

内核 5.15.0-86 中存在的 `nvme` 驱动竞争条件导致了以下特定场景：

1. NVMe 驱动通过 I/O 队列 QID 2 提交了一个读取命令
2. 命令在设备上完成，设备在向量 17 上触发 MSI-X 中断
3. 由于中断处理器中的竞争条件（向量正在被重新分配给不同的队列），中断没有传递到任何 CPU
4. 该命令在驱动的内部跟踪表中被标记为"在途"
5. 驱动从未收到完成通知，因此从未释放该命令槽位

### 4. 块层背压

Linux 块层实现了一个基于标签的系统来限制每个队列的在途 I/O 请求数。每个请求被分配一个"标签"（标识符）。当所有标签都被永远不会完成的命令消耗完时，块层无法向该队列发出任何新的 I/O 请求。

I/O 队列完全阻塞后：

- 随后的每个 `read()`、`write()` 和 `fsync()` 系统调用都会进入 `io_schedule()` 等待标签可用
- 没有标签会变得可用，因为丢失的命令永远不会完成
- 所有等待 I/O 的进程进入 D 状态并永久停留

### 5. Hung Task 看门狗触发但无法恢复

120 秒后，hung task 看门狗记录了"blocked for more than 120 seconds"的消息。但看门狗无法修复问题，因为：

- 保护 I/O 队列的内核锁被卡住的 NVMe 命令持有
- hung task 检测器只记录警告——它不会终止进程（对于 D 状态来说那是不安全的）
- 即使它试图杀死一个 D 状态进程，内核也会拒绝，因为进程处于不可中断睡眠状态

### 6. 没有驱动重置就无法恢复

摆脱这种状态的唯一办法是：

- NVMe 驱动检测到超时并重置控制器（这可能自动发生）
- 管理员通过 sysfs 触动手动驱动重置
- 完全重启系统

在这次故障中，驱动的自动超时检测也失败了，因为 MSI-X 向量耗尽导致超时检查本身也挂死了——这是一个级联故障。

---

## 解决方案

### 紧急响应

服务器处于生产环境中，有活跃用户。需要立即恢复。

#### 尝试一：NVMe 驱动重置

```bash
echo 1 > /sys/class/nvme/nvme0/device/reset_controller
```

或者使用 `nvme-cli` 工具：

```bash
nvme reset /dev/nvme0n1
```

如果重置命令成功，你会看到内核消息：

```
nvme nvme0: resetting controller
nvme nvme0: 32/0/0 default/read/poll queues
nvme nvme0: new controller alive
```

重置后，块层恢复，挂起的 I/O 完成（或失败），D 状态进程恢复正常。

在本次故障中，重置命令也挂死了，因为内核太忙无法处理 sysfs 写入。驱动的内部状态机处于不可恢复的状态。

#### 尝试二：紧急重启

当驱动重置失败时，只有重启才能恢复系统。

重启挂死系统最安全的方法是使用 SysRq 键：

```bash
echo b > /proc/sysrq-trigger
```

如果 `/proc/sysrq-trigger` 不存在或返回"Operation not permitted"，先启用 SysRq：

```bash
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

`echo b` SysRq 命令执行以下操作：

1. 同步所有已挂载的文件系统（如果可能）
2. 以只读方式重新挂载文件系统（如果可能）
3. 触发 CPU 硬重启

这比按物理复位按钮安全得多，因为内核有机会先同步文件系统。

重启后，验证：

```bash
nvme list
```

```
Node                  SN                   Model                                    Namespace Usage                      Format           FW Rev
----------------     --------------------  ----------------------------------------  --------- -------------------------- ---------------- --------
/dev/nvme0n1         PHL0123456789         INTEL SSDPE2KX040T8                      1           4.00  TB /   4.00  TB      512   B +  0B   QDV1B5
```

```bash
dmesg | grep -i nvme | tail -10
```

```
nvme nvme0: 32/0/0 default/read/poll queues
nvme nvme0: new controller alive
nvme nvme0: NVME_CMD_SCSI 0x00, I/O 0 QID 0
```

并确认没有 D 状态进程残留：

```bash
ps -eo stat,pid,comm | grep D
```

输出应该为空。

### 长期预防

恢复服务器后，实施以下措施以防止再次发生：

#### 1. 更新内核和 NVMe 驱动

该 bug 在内核 5.15.0-92 中修复（在 5.15.0-100+ 中完全解决）。升级到包含 NVMe MSI-X 竞争条件修复的内核版本。

对于 Ubuntu 22.04：

```bash
sudo apt update
sudo apt install --install-recommends linux-generic-hwe-22.04
sudo reboot
```

#### 2. 减少 NVMe 超时时间

默认的 NVMe 超时是管理命令 60 秒，I/O 命令 30 秒。降低这些值以便更快检测挂死：

添加到内核启动参数（在 `/etc/default/grub` 中）：

```
GRUB_CMDLINE_LINUX_DEFAULT="... nvme_core.admin_timeout=30 nvme_core.io_timeout=30"
```

然后：

```bash
sudo update-grub
sudo reboot
```

#### 3. 创建 NVMe 模块配置

创建 `/etc/modprobe.d/nvme.conf`：

```
options nvme_core io_timeout=30 admin_timeout=30 max_host_queues=8
```

`max_host_queues=8` 参数限制 I/O 队列数量，降低达到 MSI-X 向量限制的概率（代价是部分并行性）。

#### 4. 增加 Hung Task 阈值

默认的 120 秒 hung task 超时对于高 I/O 延迟的系统来说可能过于激进。增加它：

```bash
sysctl -w kernel.hung_task_timeout_secs=300
```

使其永久生效：

```bash
echo "kernel.hung_task_timeout_secs=300" >> /etc/sysctl.d/99-hung-task.conf
```

#### 5. 建立监控

监控 D 状态进程数量。一个简单的脚本：

```bash
#!/bin/bash
# /usr/local/bin/check_d_state.sh
D_COUNT=$(ps -eo stat | grep -c "^D")
if [ "$D_COUNT" -gt 5 ]; then
    echo "CRITICAL: $D_COUNT processes in D state | d_state=$D_COUNT"
    exit 2
elif [ "$D_COUNT" -gt 2 ]; then
    echo "WARNING: $D_COUNT processes in D state | d_state=$D_COUNT"
    exit 1
fi
echo "OK: $D_COUNT processes in D state | d_state=$D_COUNT"
```

对于基于 Prometheus 的监控，添加节点级指标：

```
node_process_state{state="D"}
```

配置告警：如果 5 个以上进程处于 D 状态持续 5 分钟以上，触发告警。

#### 6. 配置内核 Crash Dump

设置 kdump，以便在 hung task 看门狗检测到致命挂死时捕获内核内存：

```bash
sudo apt install linux-crashdump
sudo dpkg-reconfigure linux-crashdump
```

然后配置 `hung_task_panic`：

```bash
sysctl -w kernel.hung_task_panic=1
```

这会使内核在检测到 hung task 时 panic，触发 kdump 捕获崩溃转储用于事后分析。

#### 7. 文档化恢复流程

将升级路径文档化：

| 步骤 | 操作 | 命令 |
|------|------|------|
| 1 | 检查 D 状态进程 | `ps -eo pid,stat,wchan,comm \| grep "^ *[0-9] D"` |
| 2 | 检查内核栈 | `cat /proc/<pid>/stack` |
| 3 | 检查 NVMe 驱动状态 | `cat /sys/class/nvme/nvme0/device/state` |
| 4 | 检查 NVMe 错误日志 | `nvme error-log /dev/nvme0n1` |
| 5 | 重置 NVMe 控制器 | `echo 1 > /sys/class/nvme/nvme0/device/reset_controller` |
| 6 | 紧急重启 | `echo b > /proc/sysrq-trigger` |
| 7 | 验证恢复 | `ps -eo stat,pid,comm \| grep D` |

---

## 经验教训

### 做得好的方面

- 尽管系统几乎挂死，SSH 仍然可用。内核调度器仍然给了 SSH 守护进程 CPU 时间，使得远程诊断成为可能。
- 内核 hung task 消息提供了清晰的调用跟踪，直接指向 I/O 层。
- NVMe 错误日志在重启后仍然保留，使事后分析成为可能。

### 可以做得更好的方面

- **没有配置 kdump。** 当驱动进入无限循环时，没有崩溃转储来分析确切的代码路径。如果配置了 kdump，根因分析可以在几小时内完成，而不是几天。
- **监控盲区。** D 状态进程数量没有被监控。如果监控了，告警会在故障发生后 120 秒（hung task 看门狗首次触发时）发出，而不是 15 分钟后 TCP 连接开始超时。
- **没有调整超时参数。** NVMe 驱动使用默认超时值。将 I/O 超时从 30 秒减少到 15 秒不会阻止挂死，但会让驱动更快检测到丢失的中断并启动恢复。
- **单点故障。** PostgreSQL 数据目录和 Redis 追加写日志位于同一个 NVMe 设备上。一个驱动 bug 同时干掉了数据库和缓存。将它们分离到不同的存储设备（或至少不同的 NVMe 命名空间）可以限制故障半径。

### 运维团队的关键要点

1. **D 状态并不总是无需重启就能恢复的。** 虽然许多 D 状态挂死会自行解决（例如慢磁盘最终完成 I/O），但驱动级别的 bug 可以使 D 状态成为永久性的。准备好执行紧急重启。

2. **SysRq 是你的朋友。** Magic SysRq 键（`echo b > /proc/sysrq-trigger`）比硬断电安全得多。它给内核同步文件系统的机会。

3. **监控 D 状态进程数量。** 单个 D 状态进程通常是正常的（数据库检查点进程、内核 worker）。五个或更多是危险信号。十个或更多是危机。

4. **调优内核 hung task 参数。** 默认的 120 秒对于延迟敏感的系统来说可能太长了。根据你的工作负载调整它。

5. **存储驱动 bug 罕见但灾难性。** NVMe 驱动通常非常稳定，但和所有内核代码一样，它也有 bug。当它们被触发时，整个 I/O 栈可能冻结。冗余存储路径和定期的内核更新是你最好的防御。

---

## 总结

### 故障时间线

| 时间 (UTC+8) | 事件 |
|-------------|------|
| 14:37:00 | PostgreSQL 自动清理在一个 120 GB 的表上启动 |
| 14:37:12 | NVMe 驱动 MSI-X 竞争条件触发；第一个完成中断丢失 |
| 14:37:15 | 块层标签耗尽；所有新 I/O 被阻塞 |
| 14:37:20 | 30+ 个进程进入 D 状态；负载平均值飙升 |
| 14:38:00 | 首个监控告警触发（Node.js 应用健康检查失败） |
| 14:39:12 | Hung task 看门狗触发；dmesg 记录被阻塞的任务 |
| 14:42:00 | SSH 诊断开始 |
| 14:45:00 | NVMe 驱动重置尝试——挂死 |
| 14:46:15 | 发起紧急 SysRq 重启 |
| 14:48:30 | 服务器恢复在线；PostgreSQL 通过 WAL 重放完成恢复 |
| 15:00:00 | 所有服务恢复 |

### 命令参考表

| 诊断目标 | 命令 |
|---------|------|
| 列出 D 状态进程 | `ps -eo pid,stat,wchan,comm \| grep "^ *[0-9] D"` |
| 统计 D 状态进程数量 | `ps -eo stat \| grep -c "^D"` |
| 查看进程内核栈 | `cat /proc/<pid>/stack` |
| 查看进程阻塞的内核函数 | `cat /proc/<pid>/wchan` |
| 查看所有 D 状态进程的内核栈 | `for p in $(ps -eo pid,stat \| awk '/D/ {print $1}'); do echo "=== PID $p ==="; cat /proc/$p/stack 2>/dev/null; done` |
| 检查内核 hung task 消息 | `dmesg \| grep -i "hung_task\|blocked for more than"` |
| 检查 NVMe 设备状态 | `cat /sys/class/nvme/nvme0/device/state` |
| 检查 NVMe 中断分配 | `cat /proc/interrupts \| grep nvme` |
| 检查 NVMe 错误日志 | `nvme error-log /dev/nvme0n1` |
| 重置 NVMe 驱动 | `echo 1 > /sys/class/nvme/nvme0/device/reset_controller` 或 `nvme reset /dev/nvme0n1` |
| 安全紧急重启 | `echo b > /proc/sysrq-trigger` |
| 配置 hung task 超时 | `sysctl -w kernel.hung_task_timeout_secs=300` |
| 查看块设备统计 | `cat /sys/block/nvme0n1/stat` |

### 最后的话

Linux D 状态是系统管理中最常被误解的进程状态之一。它不是一个通常意义上的"卡住"状态——它是内核在说"我已经承诺了这个 I/O 操作，我无法回头"。当它正常工作时，它保护数据完整性。当驱动 bug 使其成为永久状态时，它可以让整个服务器瘫痪。

最重要的要点：**你无法杀死一个 D 状态进程。** 不要浪费时间尝试 `kill -9`。找到它正在等待的 I/O 设备，修复设备或驱动，然后进程才会恢复。
