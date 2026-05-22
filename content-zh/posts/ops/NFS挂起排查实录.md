---
title: "NFS 挂起排查实录——当所有进程陷入 D 状态"
date: 2026-05-22
weight: 100150
slug: "nfs-hang-d-state-troubleshooting"
tags: ["nfs", "storage", "d-state", "linux", "troubleshooting"]
categories: ["存储"]
description: "NFS 服务器宕机后客户端全部进程卡死，从 SSH 挂起到定位 D 状态进程，到 umount -f 强制恢复的完整过程"
keywords: "nfs hang, d state, uninterruptible sleep, nfs hard mount, process hung"
draft: false
featured: true
cover:
  image: "/images/nfs-hang-banner.svg"
  caption: "NFS 挂起导致进程 D 状态排查"
---

# NFS 挂起排查实录——当所有进程陷入 D 状态

## 问题现象

周五下午 4 点半，告警突然炸了。

一个 Java 应用服务器节点完全失联——不是"响应慢"，是彻底连不上。监控面板显示该节点上的所有服务探针全部超时。

SSH 试一下：

```bash
ssh app-server-01
```

```
ssh: connect to host app-server-01 port 22: Connection timed out
```

连 SSH 都进不去。这就不是应用问题了，是整个操作系统层面出了问题。

再看看其他节点的监控——这个节点上跑的应用在其他节点也有副本，但副本也陆续开始报错。日志显示：

```
java.io.IOException: No such device or address
   at java.base/sun.nio.ch.FileDispatcherImpl.write0(Native Method)
```

另一个服务报：

```
Error syncing pod, skipping: failed to "PrepareDynamicResources" for "app-pod" 
  with PrepareDynamicResourcesError: "rpc error: code = Internal 
  desc = wait for remote storage condition: context deadline exceeded"
```

同时，部署在该节点的 kubelet 也失联了，节点状态变成 NotReady。

```bash
kubectl get nodes
```

```
NAME            STATUS     ROLES    AGE
app-server-01   NotReady   worker   187d
app-server-02   Ready      worker   187d
app-server-03   Ready      worker   187d
```

**影响**：该节点上 20+ 个 Pod 全部不可用，部分 Pod 无法被调度到其他节点（用了 local PV / hostPath），核心业务受损。更麻烦的是——K8S 无法正常驱逐该节点上的 Pod，因为 kubelet 本身也卡住了。

## 排查过程

### 错误尝试 1：尝试 SSH 进节点

SSH 连不上，先试带 -v 看看卡在哪一步：

```bash
ssh -vvv app-server-01
```

```
debug1: Connecting to app-server-01 [10.0.0.10] port 22.
debug1: Connection established.
debug1: identity file /home/user/.ssh/id_rsa type 0
debug1: Local version string SSH-2.0-OpenSSH_8.2p1
# 然后就卡住了，没有任何输出
```

TCP 连接能建立（说明内核网络栈还在工作），但 SSH 在认证阶段卡死了——因为 SSH 的 PAM 模块要读取 `/var/run` 或 `/home` 里的文件（这些路径挂载了 NFS）。

**踩坑点**：这台机器的 `/home` 是 NFS 挂载的（用户目录统一存在 NAS 上）。SSH 登录时 PAM 要读用户的 authorized_keys，这个文件在 NFS 上——NFS 挂了，SSH 就在那儿等着了。这不是 SSH 的问题，也不是网络的问题，是 NFS 把所有 I/O 都卡死了。

后来是通过带外管理（BMC/iLO/iDRAC）才进到机器的：

```bash
# 通过 IPMI/KVM 连上去后，第一件事看进程状态
ps aux | grep "^[RD]"
```

```
USER       PID %CPU %MEM    VSZ   RSS STAT START   TIME COMMAND
root      1234  0.0  0.1      0     0 D    Apr22   0:01 [kworker/0:0]
app       2345  0.0  0.2 423456 12345 D    15:30   0:15 java -jar app.jar
app       2346  0.0  0.1 123456  7890 D    15:31   0:10 nginx: worker process
root      3456  0.0  0.0      0     0 D    15:32   0:00 [nfsv4.0]
root      4567  0.0  0.0      0     0 D    15:33   0:00 [kworker/u4:0]
```

一列 **D 状态** 进程——Uninterruptible Sleep。

### 错误尝试 2：以为被 DDoS 或 OOM

看到一堆 D 状态进程，第一反应是不是资源耗尽。

```bash
# 先排除 OOM
dmesg | grep -i "out of memory"
```

没有 OOM 记录。Swap 也正常：

```bash
free -h
```

```
              total        used        free      shared  buff/cache   available
Mem:           31Gi        18Gi         2Gi       1.0Gi        11Gi        12Gi
Swap:           2Gi       200Mi       1.8Gi
```

内存和 CPU 都还充裕。

```bash
# 系统负载
uptime
```

```
 16:45:00 up 187 days,  3:12,  0 users,  load average: 45.12, 30.21, 15.08
```

Load average 45——这台机器只有 16 核。但这不代表 CPU 忙，D 状态进程也算在 load average 里。U 状态的进程其实很少。

### 错误尝试 3：试图 kill -9 干掉卡住的进程

直觉告诉我"先杀了进程再说"：

```bash
kill -9 2345
```

```
-bash: kill: (2345) - Operation not permitted
```

等等，我是 root 还不让杀？

```bash
kill -9 3456
```

```
-bash: kill: (3456) - Operation not permitted
```

**踩坑点**：D 状态（Uninterruptible Sleep）意味着进程正在执行一个内核态操作（比如 NFS I/O），这个操作不能被中断。`kill -9` 发送的是 SIGKILL，但 SIGKILL 在进程处于 D 状态时**不会立即生效**——内核必须等进程回到用户态才能处理信号，而进程被 NFS I/O 堵在 D 状态回不来。

这就是 D 状态最坑的地方——**D 状态的进程是不可杀的**。`kill -9` 不会报错，信号已经标记上了，但进程不会死，直到内核 I/O 操作完成或超时。

对于 NFS 的场景，如果 mount 选项是 `hard`（默认），NFS 客户端会无限重试，**永远不超时**——意味着 D 状态进程永远不会恢复。

```bash
# 查看 mount 选项
mount | grep nfs
```

```
10.0.0.100:/data on /mnt/nfs type nfs4 (rw,relatime,vers=4.0,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.0.0.10,local_lock=none,addr=10.0.0.100)
```

明确写着 **`hard`**——意味着 NFS 客户端永远不会放弃重试。只要 NFS 服务器不恢复，这些进程会永远等下去。

### 真正的发现：NFS 服务器挂了

检查 NFS 服务器：

```bash
# 从另一个节点 ping NFS 服务器
ping 10.0.0.100
```

```
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
...
```

NFS 服务器完全失联。

```bash
# 检查卡住的 NFS 客户端状态
cat /proc/fs/nfsfs/servers
```

```
NV SERVER   PORT USE HOSTNAME
v4 0a000064  801   1 10.0.0.100
```

```bash
# 看具体哪些进程在等 NFS
cat /proc/2345/stack
```

```
[<ffffffffc06b3a40>] nfs4_wait_bit_killable+0x20/0x60 [nfsv4]
[<ffffffffc069e2a0>] __rpc_execute+0x250/0x3c0 [sunrpc]
[<ffffffffc069d3c0>] rpc_run_task+0x100/0x120 [sunrpc]
[<ffffffffc06b4810>] nfs4_call_sync_private+0x90/0xc0 [nfsv4]
[<ffffffffc06b8890>] _nfs4_do_open+0x390/0x650 [nfsv4]
[<ffffffffc06b8ff0>] nfs4_open+0xa0/0x120 [nfsv4]
[<ffffffffc069ae90>] nfs4_file_open+0x60/0xc0 [nfsv4]
[<ffffffff8121b340>] do_dentry_open+0x140/0x2e0
```

可以看到调用栈全部卡在 `nfs4_wait_bit_killable` → `__rpc_execute` → NFS RPC 层在等服务器响应。

```bash
# D 状态进程总数统计
ps -eo stat | grep -c "^D"
```

```
23
```

23 个 D 状态进程。包括 Java 主进程、Nginx worker、sshd、kubelet，甚至一些内核 worker。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | NFS 存储服务器 10.0.0.100 宕机（后续排查是电源故障导致重启后未正常挂载阵列） |
| 传导机制 | NFS `hard` 挂载模式下，客户端进程 I/O 陷入 D 状态，无限重试不可中断 |
| 影响扩散 | 关键系统进程（sshd、kubelet）也因访问 NFS 挂载点卡死，节点完全失联 |
| 恢复阻碍 | D 状态进程不可 kill，只等 NFS 恢复或重启系统 |

**为什么 NFS `hard` 这么危险**：

`hard` vs `soft` 的区别：

| 挂载选项 | NFS 服务器挂起时 |
|----------|------------------|
| `hard`（默认） | 进程无限重试，始终 D 状态，永远不会返回错误给应用 |
| `soft` | 重试 `retrans` 次后超时返回 I/O 错误，应用可以 catch |

很多发行版和 Kubernetes 存储插件的默认值就是 `hard`。`hard` 的好处是数据不会丢（不会悄悄返回写失败），但代价是——**NFS 一挂，整个机器跟着挂**。

## 解决方案

### 方案 A：等待 NFS 服务器恢复（被动，但不一定能等来）

如果 NFS 服务器能快速恢复，D 状态进程会自己活过来：

```bash
# NFS 服务器恢复后，检查客户端
mount | grep nfs
# 挂载应该还在
ps aux | grep " D"
# D 状态进程会逐渐减少
```

但这次 NFS 服务器是电源故障，阵列恢复需要时间——我们不能干等。

### 方案 B：强制卸载 NFS 挂载（主动恢复，但有风险）

```bash
# 先试试 umount -f（强制卸载）
umount -f /mnt/nfs
```

```
umount: /mnt/nfs: target is busy
```

```bash
# 找到占用 NFS 的进程
fuser -m /mnt/nfs
```

```
/mnt/nfs:           1234 2345 2346 3456 4567
```

```bash
# 用 umount -l（lazy unmount）
umount -l /mnt/nfs
```

`umount -l` 会立刻从文件系统树中移除挂载点，让新的访问无法再走到 NFS。但**已经在 D 状态里的进程不会恢复**——它们还在等内核 NFS 层的 RPC 回复。

```bash
# 如果 lazy unmount 后进程依然卡死，需要重启（见方案 C）
```

**注意**：`umount -l` 在某些内核版本上可能导致后续无法正常重启，因为内核 NFS 状态机已经混乱了。尽量和方案 C 一起用。

### 方案 C：重启系统（最终手段，但必须做对）

因为 SSH 连不上，重启只能通过带外管理（BMC）：

```bash
# iDRAC / IPMI 工具
ipmitool -H bmc-ip -U admin -P pass chassis power cycle
```

但**重启不一定能顺利**——因为系统在 unmount 阶段会尝试写回 NFS，而 NFS 还挂着，系统可能会卡在 shutdown 阶段。

```bash
# 在 BMC 控制台执行（可能不成功）
reboot -f
```

如果 `reboot -f` 卡住，只能物理断电再开机：

```bash
# IPMI 硬重置
ipmitool -H bmc-ip -U admin -P pass chassis power off
# 等待 30 秒
ipmitool -H bmc-ip -U admin -P pass chassis power on
```

**真实情况**：这台机器最后是靠 BMC 硬断电重启的。重启后检查：

```bash
# 确认所有 D 状态进程消失
ps -eo stat | grep -c "^D"
```

```
0
```

```bash
# 正常挂载检查
mount | grep nfs
```

没有 NFS 挂载了（之前 lazy umount 过了）。

```bash
# kubelet 恢复
kubectl get nodes
```

```
NAME            STATUS   ROLES    AGE
app-server-01   Ready    worker   187d
```

✅ 节点恢复正常。

### 验证恢复

```bash
# 1. 确认无 D 状态进程
ps -eo stat,pid,comm | grep "^D"
# 无输出

# 2. 确认 kubelet 状态
systemctl status kubelet
# active (running)

# 3. 确认 Pod 恢复
kubectl get pods -o wide | grep app-server-01
# 全部 Running/Ready

# 4. 确认应用正常
curl http://app-service:8080/health
# 200 OK
```

### 长期修复——防止再被 NFS 坑

```bash
# 1. 使用 soft 挂载 + intr（内核 2.6+ 后 intr 已弃用，但 soft 保留）
# 修改 /etc/fstab
10.0.0.100:/data  /mnt/nfs  nfs4  soft,timeo=30,retrans=3,noac  0 0
```

```
mount -o remount /mnt/nfs
```

**注意**：`soft` 不是万能的。对于数据库等需要强一致性的场景，`soft` 可能导致静默数据损坏（写操作返回成功但实际没落盘）。在这些场景下用 `hard` 是合理的——但要接受"NFS 挂则机器挂"的前提。

**更安全的替代方案**：

| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| `soft` + 应用重试 | 无状态服务、缓存 | NFS 挂了不影响机器 | 可能丢写操作 |
| `hard` + NFS HA | 数据库、有状态服务 | 数据一致性保证 | 需要 NFS 高可用架构 |
| 不使用 NFS，走对象存储 API | 新系统 | 故障隔离 | 应用改造大 |
| K8S CSI + 本地存储 | K8S 工作负载 | 故障域隔离 | 不是所有场景适用 |

```bash
# 2. 增加内核参数——减少 D 状态进程导致的 load 误解
# 不直接解决问题，但避免负载告警误报
# 监控层面区分：CPU load vs D state count

# 3. 部署 NFS HA 方案（多活/主备）
# NFS 服务端高可用，避免单点故障
# DRBD + Pacemaker / GlusterFS / CephFS

# 4. 监控 NFS 连通性
# 定期探测 NFS 服务器，提前告警
while true; do
  timeout 5 ls /mnt/nfs/.health 2>/dev/null || echo "NFS down: $(date)" >> /var/log/nfs_watchdog.log
  sleep 60
done
```

## 教训总结

1. **D 状态的进程是不可杀的。** 永远不要指望 `kill -9` 能干掉一个 D 状态进程。你杀的只是进程要处理的一个信号，但进程永远回不到用户态去处理它。了解 D 状态和 Z 状态（僵尸）的区别——前者杀不死，后者可以 wait 掉。

2. **`nfs hard` 选项是双刃剑。** 它能保证数据不丢（不会静默返回写失败），但代价是"如果 NFS 挂了，连 SSH 都进不去"。关键系统最好不要把 `/home`、`/var/log` 这些路径挂载到 NFS 上——否则 NFS 一挂，你连排查的工具都用不了。

3. **最恐怖的排查场景是什么？** 不是错误日志几千行，而是你 SSH 都连不上机器。这次如果没有 BMC，我们连怎么回事都搞不清楚。对于生产环境，**带外管理不是选项，是必需品**。

4. **`/proc/<pid>/stack` 是排查 D 状态的神器。** `kill -9` 杀不死进程的时候，别试第二遍。去看 `/proc/<pid>/stack`，看懂进程卡在哪个内核函数上。`nfs4_wait_bit_killable` 一看就知道是 NFS 的问题。

5. **K8S 环境下 NFS 的问题会放大。** kubelet、CRI shim、监控 agent 都可能依赖节点上的文件系统。一个 NFS 挂载卡住，可能导致 kubelet 失联、节点 NotReady、Pod 无法驱逐——连锁反应。如果你的 K8S 用 NFS 做存储，一定要做好 NFS 高可用，并且不要把关键系统路径放在 NFS 上。
