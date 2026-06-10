---
title: "systemd 单元启动超时导致服务不可用"
date: 2026-06-11
weight: 100500
slug: "systemd-unit-timeout-service-down"
tags: ["linux", "systemd", "system", "troubleshooting", "service"]
categories: ["Troubleshooting"]
description: "一起 systemd 单元超时事故——NFS 挂载在启动时卡死导致多个关键服务在默认的 90 秒 TimeoutStartSec 内启动失败，生产服务器部分离线"
keywords: "systemd 单元超时, systemd TimeoutStartSec, journalctl -u 服务, systemd 依赖,  Linux 服务启动故障排查"
draft: false
featured: true
cover:
  image: ""
  caption: "systemd 单元超时排查"
---

## 常见搜索词

- systemd 服务重启后超时
- systemctl status 显示超时失败
- NFS 挂载导致 systemd 启动卡死
- fstab 缺少 _netdev 选项
- TimeoutStartExceeded systemd
- remote-fs.target 依赖失败
- Nginx 重启后 systemd 超时无法启动
- systemd 单元启动请求过于频繁
- journalctl -u 服务超时
- systemd 依赖失败导致服务不可用

## 故障经过

### 环境

- **操作系统**：Ubuntu 22.04 LTS
- **PostgreSQL**：15（运行在本地 SSD 上）
- **Nginx**：1.24（反向代理，日志写入 NFS 挂载）
- **Tomcat**：9（从 NFS 共享部署应用）
- **NFS 挂载**：Nginx 访问日志和 Tomcat 部署包共享存储，由专用 NAS 设备提供
- **计划重启**：内核更新需要重启，安排在凌晨 02:00

### 时间线

凌晨 02:00，服务器按计划执行内核更新后的优雅关机。硬件 POST 顺利完成，GRUB 加载了新内核。但接下来发生了一系列连锁故障，导致生产服务器部分失明长达 15 分钟。

### 症状

当班工程师在收到自动健康检查告警后登录服务器时发现：

- **SSH**：正常——服务器响应迅速
- **PostgreSQL**：运行中并可接受连接——本地数据未受影响
- **Nginx**：返回 HTTP 502 Bad Gateway——没有可用的上游
- **Tomcat**：端口 8080 和 8443 均无响应——完全无法访问
- **系统监控**：Grafana 面板显示 Nginx 和 Tomcat 指标出现空白

乍一看，服务器似乎很正常。SSH 可用，PostgreSQL 正常。但承载用户流量的两个服务全部宕机。这就是典型的"服务器看起来在线但实际上并不在线"的场景，使得 systemd 超时故障尤其危险。

## 背景

### systemd 启动序列

使用 systemd 的现代 Linux 系统遵循结构化的启动流程。内核初始化硬件后，systemd（PID 1）启动初始目标（default.target，通常别名指向 graphical.target 或 multi-user.target）。Systemd 通过**单元**（服务 .service、挂载点 .mount、设备 .device、套接字 .socket、目标 .target）的有向无环图来解析依赖关系。

本次事故相关的启动顺序：

1. **local-fs.target** — 挂载本地文件系统
2. **remote-fs.target** — 挂载网络文件系统（NFS、CIFS）
3. **network.target** — 基本网络可用
4. **multi-user.target** — 启动所有服务进入正常运行状态

像 Nginx 和 Tomcat 这样的关键服务在依赖网络存储时，通常会在单元文件中声明 `After=remote-fs.target`。这意味着它们会等待 remote-fs.target 完成之后才会启动。

### 服务启动超时机制

每个 systemd 服务单元都有一个 `TimeoutStartSec` 指令。在大多数发行版（包括 Ubuntu 22.04）中，其默认值为 **90 秒**。这个参数控制 systemd 等待服务报告启动成功的时长。

但超时并不仅限于服务单元。挂载单元（.mount）也继承了超时行为。当一个挂载单元卡死时（例如 NFS 服务器不可达），systemd 会等待 `TimeoutStartSec` 的时长，然后将该挂载标记为失败。

关键指令：

- **TimeoutStartSec=90** — 服务启动超时默认值
- **TimeoutStopSec=90** — 服务停止超时默认值
- **DefaultTimeoutStartSec=90** — systemd 编译时默认值
- **TimeoutStartSec=infinity** — 禁用超时（生产环境不推荐）

### systemd 依赖图

理解单元之间的关系至关重要：

- **`After=`** — 仅排序；本单元在指定单元之后启动，但指定单元失败不影响本单元启动
- **`Requires=`** — 强依赖；如果指定单元失败，本单元将被停止或停用
- **`Wants=`** — 弱依赖；systemd 尝试启动指定单元，但允许失败
- **`PartOf=`** — 如果指定单元被停止或重启，本单元跟随相同操作

在我们的案例中，Nginx 的单元文件很可能包含 `After=remote-fs.target`，可能还包含 `Requires=remote-fs.target`。当 remote-fs.target 中的 NFS 挂载超时后，整个目标进入失败状态，连锁影响到依赖它的服务。

### fstab 挂载选项与 systemd

`/etc/fstab` 文件在启动时由 systemd-fstab-generator 解析，生成对应的 `.mount` 单元。挂载选项直接影响 systemd 的行为：

| 选项 | 作用 |
|--------|--------|
| `_netdev` | 将挂载标记为网络文件系统；systemd 等待网络就绪后再尝试挂载 |
| `nofail` | 如果挂载失败，systemd 继续执行，不会标记为失败——对非关键网络挂载至关重要 |
| `x-systemd.automount` | 创建 automount 单元而非 mount 单元；文件系统在首次访问时挂载，而非启动时 |
| `x-systemd.device-timeout=30s` | 设备出现的超时时间，超时后 systemd 放弃等待 |
| `defaults` | 无特殊 systemd 行为；挂载被视为本地文件系统依赖 |

缺少 `_netdev` 是根因。没有 `_netdev`，systemd 会将 NFS 挂载视为本地文件系统依赖，在启动序列中作为 `local-fs.target` 的一部分或早期依赖等待其完成。

## 排查过程

以下是完整的排查步骤。每条命令都揭示了问题的一个片段。

### 第一步：检查系统整体状态

```bash
systemctl list-units --failed
```

输出：
```
UNIT                        LOAD   ACTIVE SUB    DESCRIPTION
var-log-nginx.mount         loaded failed failed /var/log/nginx
nginx.service               loaded failed failed Nginx Web Server
tomcat.service               loaded failed failed Apache Tomcat
remote-fs.target            loaded failed failed Remote File Systems
```

四个失败的单元。这个模式立即引起了警觉——三个服务和一个挂载点全部处于失败状态。`var-log-nginx.mount` 指向 NFS 共享。

### 第二步：检查各个服务状态

```bash
systemctl status nginx
```

```
● nginx.service - Nginx Web Server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: failed (Result: timeout) since Thu 2026-06-11 02:03:30 UTC; 12min ago
       Docs: man:nginx(8)
    Process: 892 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 893 ExecStart=/usr/sbin/nginx -g daemon on; (code=exited, status=0/SUCCESS)
   Main PID: 893 (code=exited, status=0/SUCCESS)
        CPU: 23ms

Jun 11 02:01:59 prod-web systemd[1]: Starting Nginx Web Server...
Jun 11 02:03:30 systemd[1]: nginx.service: Start request repeated too quickly.
Jun 11 02:03:30 systemd[1]: nginx.service: Failed with result 'timeout'.
Jun 11 02:03:30 systemd[1]: Failed to start Nginx Web Server.
```

关键发现：Nginx 的 ExecStartPre（配置测试）和 ExecStart 均以代码 0（成功）退出。然而 systemd 报告了超时。这很不寻常——通常超时意味着主进程未在窗口期内就绪。但这里进程启动并成功退出了。"start request repeated too quickly"消息暗示了依赖问题：systemd 正在重试 Nginx，因为某个依赖（NFS 挂载）最终失败了，systemd 在反复尝试满足依赖链。

```bash
systemctl status tomcat
```

```
● tomcat.service - Apache Tomcat
     Loaded: loaded (/etc/systemd/system/tomcat.service; enabled; vendor preset: enabled)
     Active: failed (Result: dependency) since Thu 2026-06-11 02:01:59 UTC; 14min ago
       Docs: https://tomcat.apache.org
    Process: 845 ExecStart=/opt/tomcat/bin/startup.sh (code=exited, status=0/SUCCESS)
   Main PID: 845 (code=exited, status=0/SUCCESS)

Jun 11 02:01:30 prod-web systemd[1]: Starting Apache Tomcat...
Jun 11 02:01:59 prod-web systemd[1]: tomcat.service: Dependency failed, entering failed status.
Jun 11 02:01:59 prod-web systemd[1]: tomcat.service: Failed with result 'dependency'.
Jun 11 02:01:59 prod-web systemd[1]: Failed to start Apache Tomcat.
```

"dependency"结果是最关键的线索。Tomcat 并非自身失败——它所依赖的某个组件先失败了。

### 第三步：检查启动日志

```bash
journalctl -b -1 | grep -i "timeout\|failed\|error" | tail -30
```

```
Jun 11 02:00:45 prod-web kernel: NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:00:45 prod-web kernel: NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:01:05 prod-web systemd[1]: var-log-nginx.mount: Mount process timed out
Jun 11 02:01:05 prod-web systemd[1]: var-log-nginx.mount: Failed with result 'timeout'
Jun 11 02:01:05 prod-web systemd[1]: Failed to mount /var/log/nginx
Jun 11 02:01:59 prod-web systemd[1]: remote-fs.target: Found dependency on var-log-nginx.mount
Jun 11 02:01:59 prod-web systemd[1]: remote-fs.target: Job verilog-nginx.mount/start failed with result 'timeout'
Jun 11 02:01:59 prod-web systemd[1]: remote-fs.target: Triggering dependency failures
Jun 11 02:03:30 prod-web systemd[1]: nginx.service: Start request repeated too quickly
Jun 11 02:03:30 prod-web systemd[1]: nginx.service: Failed with result 'timeout'
Jun 11 02:03:30 prod-web systemd[1]: Failed to start Nginx Web Server
```

NFS 内核模块报告了错误 -512（在内核上下文中表示服务器不可达）。随后挂载单元超时。systemd 随后花费了额外时间重试，然后标记目标和其依赖的服务为失败。

### 第四步：检查单元依赖

```bash
systemctl list-dependencies nginx
```

```
nginx.service
● ├─system.slice
● ├─nginx.service
● └─remote-fs.target
●   └─var-log-nginx.mount
```

这确认了依赖链。Nginx 依赖 `remote-fs.target`，而 `remote-fs.target` 又依赖 `var-log-nginx.mount`，即失败的 NFS 挂载。

```bash
systemctl list-dependencies tomcat
```

```
tomcat.service
● ├─system.slice
● ├─tomcat.service
● └─remote-fs.target
●   └─var-log-nginx.mount
```

Tomcat 有相同的依赖链。

### 第五步：排查 NFS 挂载

```bash
mount | grep nfs
```

无输出。NFS 挂载未激活。

```bash
systemctl status remote-fs.target
```

```
● remote-fs.target - Remote File Systems
     Loaded: loaded (/usr/lib/systemd/system/remote-fs.target; static)
     Active: failed (Result: dependency) since Thu 2026-06-11 02:01:59 UTC; 15min ago
       Docs: man:systemd.special(7)
```

```bash
journalctl -u remote-fs.target
```

```
Jun 11 02:00:40 prod-web systemd[1]: Reached target Remote File Systems.
Jun 11 02:00:45 prod-web systemd[1]: var-log-nginx.mount: Mount process timed out
Jun 11 02:00:45 prod-web systemd[1]: remote-fs.target: Job var-log-nginx.mount/start failed with result 'timeout'
Jun 11 02:00:45 prod-web systemd[1]: remote-fs.target: Triggering dependency failures
Jun 11 02:00:45 prod-web systemd[1]: remote-fs.target: Dependency failed, entering failed status
```

日志显示 remote-fs.target 最初已达成，但 NFS 挂载超时后立即失败。

### 第六步：检查 /etc/fstab

```bash
cat /etc/fstab | grep nfs
```

```
nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0
```

找到问题了。NFS 条目使用了 `defaults` 而非 `_netdev`。这就是铁证。

### 第七步：检查 NFS 服务器连通性

```bash
timeout 5 showmount -e nfs-server
```

```
showmount: RPC: Program not registered
```

NFS 服务器无响应。NAS 设备在维护期间已被下线，这是根本触发因素。

### 第八步：检查系统日志中的启动时 NFS 错误

```bash
grep -i "nfs\|mount" /var/log/syslog | grep -i "error\|fail\|timeout"
```

```
Jun 11 02:00:41 prod-web kernel: [   15.432] NFS: Registering the id_resolver key type
Jun 11 02:00:41 prod-web kernel: [   15.432] FS-Cache: Netfs 'nfs' registered for caching
Jun 11 02:00:42 prod-web kernel: [   16.105] NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:00:42 prod-web kernel: [   16.105] NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:00:45 prod-web systemd[1]: var-log-nginx.mount: Mount process timed out
```

内核 NFS 客户端尝试发现服务器 trunking 能力，但服务器不可达，反复返回错误 -512，直到挂载超时。

## 根因

根因链条非常清晰：

### 主要原因：fstab 中缺少 _netdev

NFS 共享的 `/etc/fstab` 条目为：

```
nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0
```

`defaults` 选项不包含 `_netdev`。systemd-fstab-generator 处理此条目时，不会将其识别为网络文件系统。没有 `_netdev`，挂载单元被视为本地挂载依赖，在启动早期网络尚未完全就绪时就开始尝试挂载。

### 次要原因：NFS 服务器不可用

提供 NFS 共享的 NAS 设备因计划维护而停机。维护窗口与服务器重启时间重叠，形成了完美风暴。

### 故障传播路径

1. **启动开始** — systemd 开始处理挂载单元
2. **NFS 挂载尝试** — `var-log-nginx.mount` 尝试挂载 `nfs-server:/exports/logs`
3. **挂载卡死** — NFS 服务器不可达；挂载命令阻塞
4. **超时到期** — `TimeoutStartSec`（默认 90 秒）过后，systemd 将 `var-log-nginx.mount` 标记为失败
5. **目标失败** — `remote-fs.target` 依赖 `var-log-nginx.mount`；进入失败状态
6. **连锁故障** — Nginx 和 Tomcat 都包含 `After=remote-fs.target` 和/或 `Requires=remote-fs.target`；无法启动
7. **systemd 重试** — systemd 尝试重启 Nginx，但依赖链已断裂；每次尝试均失败
8. **启动请求过于频繁** — systemd 检测到重复快速失败，停止尝试

### 为什么 PostgreSQL 未受影响

PostgreSQL 将数据存储在本地 SSD 上。其 systemd 单元文件没有依赖 `remote-fs.target` 或任何 NFS 挂载。数据库服务独立启动，在整个事件期间完全正常运行。

### 部分离线的假象

服务器看似健康的原因为：
- SSH 守护进程（sshd）不依赖 remote-fs.target
- PostgreSQL 无 NFS 依赖
- 基本系统服务（syslog、cron 等）运行在本地存储上
- 内核启动成功，网络功能正常

只有依赖 NFS 挂载的服务受到影响。这种选择性故障使得此类事故更难通过简单的"服务器是否在线"检查来发现。

## 解决方案

### 紧急修复

当务之急是恢复 Nginx 和 Tomcat。修复需要纠正 fstab 条目并重新挂载。

**第一步：修复 /etc/fstab**

编辑 `/etc/fstab`，更新 NFS 挂载条目：

```
# 旧的（有问题的）：
# nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0

# 新的（已修复的）：
nfs-server:/exports/logs  /var/log/nginx  nfs  _netdev,noexec,nofail,x-systemd.automount,x-systemd.device-timeout=30s  0  0
```

关键选项说明：

| 选项 | 作用 |
|--------|--------|
| `_netdev` | 告知 systemd 这是一个网络文件系统；等待网络就绪 |
| `nofail` | 如果挂载失败，systemd 继续执行，不标记为失败 |
| `x-systemd.automount` | 首次访问时挂载而非启动时挂载——对网络文件系统至关重要 |
| `x-systemd.device-timeout=30s` | 等待 NFS 服务器 30 秒后放弃 |
| `noexec` | 安全加固——防止从挂载的文件系统执行程序 |

**第二步：重新加载 systemd**

```bash
systemctl daemon-reload
```

这告诉 systemd 重新读取从 fstab 生成的单元文件。

**第三步：重启失败的服务**

```bash
systemctl restart nginx tomcat
```

**第四步：验证服务状态**

```bash
systemctl is-active nginx tomcat
```

预期输出：
```
active
active
```

**第五步：验证挂载状态**

```bash
mount | grep nfs
```

使用 `x-systemd.automount` 时，挂载可能不会立即显示。它将在首次访问时被挂载。如需强制挂载：

```bash
systemctl restart var-log-nginx.mount
```

或者直接访问挂载点：
```bash
ls -la /var/log/nginx/
```

### 长期预防措施

#### 1. 为所有网络文件系统挂载添加 nofail

审计 `/etc/fstab` 中所有引用网络文件系统（NFS、CIFS、GlusterFS 等）的条目。每个网络挂载都应包含 `nofail`，以防止挂载失败阻塞启动序列。

```bash
grep -E "nfs|cifs|smb|gluster|fuse" /etc/fstab
```

对于每个条目，在选项字段中添加 `_netdev,nofail,x-systemd.automount`。

#### 2. 对 NFS 实施 x-systemd.automount

`x-systemd.automount` 选项创建一个 systemd automount 单元而非 mount 单元。文件系统在进程首次访问挂载点时按需挂载。这有几个优点：

- 启动不会被 NFS 可用性阻塞
- 如果 NFS 服务器宕机，服务仍可启动（它们只在实际访问挂载时才会遇到 I/O 错误）
- 访问时会自动重试挂载
- 如果网络稍后可用，systemd 可以重新加载挂载

#### 3. 审查关键服务依赖

对于每个关键服务，审查其 systemd 单元文件中的不必要文件系统依赖：

```bash
systemctl cat nginx
```

查找 `After=`、`Requires=` 和 `Wants=` 指令。考虑列出的所有依赖是否对服务功能真正必要。如果某个依赖是"有更好"而非"必须"，将 `Requires=` 降级为 `Wants=`，并为相关挂载添加 `nofail`。

#### 4. 添加启动时监控

防止静默部分故障的最佳防御是监控。添加以下检查：

**Prometheus 系统启动时单元状态监控**：
```yaml
# prometheus rule
- alert: SystemdFailedUnits
  expr: node_systemd_unit_state{state="failed"} > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "在 {{ $labels.instance }} 上检测到 systemd 失败单元"
```

**自定义启动健康检查脚本**：
```bash
#!/bin/bash
# /usr/local/bin/boot-health-check.sh
# 作为 systemd oneshot 服务在 multi-user.target 之后运行

FAILED_UNITS=$(systemctl list-units --failed --no-legend --no-pager | wc -l)
if [ "$FAILED_UNITS" -gt 0 ]; then
    systemctl list-units --failed --no-legend --no-pager | \
        mail -s "Boot Health Check Failed on $(hostname)" ops@example.com
fi
```

**Grafana 面板告警** — 监控 `node_systemd_unit_state` 指标，对任何失败单元触发告警。

#### 5. 为确实需要更长时间的调整 TimeoutStartSec

某些服务需要超过 90 秒才能启动。对于这些服务，在 drop-in 配置中设置适当的 `TimeoutStartSec`，而非全局修改：

```bash
mkdir -p /etc/systemd/system/nginx.service.d
cat > /etc/systemd/system/nginx.service.d/timeout.conf << 'EOF'
[Service]
TimeoutStartSec=300s
EOF
systemctl daemon-reload
```

注意：这不是本次事故的修复方案。不修复 fstab 而仅增加超时只会推迟失败，而非预防失败。

#### 6. 在测试环境测试重启流程

在任何未来的内核更新或重启之前：

1. 在测试环境中执行分阶段重启
2. 自动验证所有关键服务在启动后的状态：
   ```bash
   # 重启后验证脚本
   for svc in nginx tomcat postgresql sshd; do
       if ! systemctl is-active --quiet "$svc"; then
           echo "失败: $svc 未运行"
           exit 1
       fi
   done
   echo "所有关键服务运行正常"
   ```
3. 模拟 NFS 服务器在启动期间故障，验证 `nofail` 和 `x-systemd.automount` 按预期工作

#### 7. 记录 systemd 单元依赖架构

创建一份活的文档，映射每个关键服务到其 systemd 依赖。这有助于工程师理解依赖失败时的影响范围：

```bash
# 为所有关键服务生成依赖图
for unit in nginx tomcat postgresql sshd; do
    echo "=== $unit ==="
    systemctl list-dependencies "$unit" --no-legend --no-pager
done
```

## 经验教训

### 1. fstab defaults 陷阱

`/etc/fstab` 中的 `defaults` 挂载选项名称具有误导性。它并不意味着"生产环境的安全默认值"——它只是内核的默认挂载标志集合（rw、suid、dev、exec、auto、nouser、async）。关键在于，它不包含 `_netdev`、`nofail` 或任何针对网络文件系统故障的 systemd 专用选项。

**教训**：对于网络文件系统挂载，永远不要使用裸 `defaults`。始终明确包含 `_netdev` 和 `nofail`。

### 2. 部分故障更难检测

一个"在线"（SSH 可达、内核运行）但缺少关键服务的服务器比完全宕机的服务器更危险。监控必须检查服务健康状态，而不仅仅是服务器可达性。

**教训**：分层监控——独立检查端口可用性、进程健康和应用程序级响应。

### 3. 超时不等于服务启动慢

当 systemd 报告超时时，直觉是增加 TimeoutStartSec。但超时通常意味着依赖已损坏，而非服务需要更多时间。始终先调查依赖链。

**教训**：看到超时时，先问"它在等待什么？"再调整定时参数。

### 4. systemd 依赖链创建爆炸半径

一个挂载单元失败可以拖垮多个不相关的服务。必须理解并定期审计 systemd 单元的依赖图。如果某个服务并不真正需要远程文件系统，就不应声明对其的依赖。

**教训**：最小化依赖链。对于非关键依赖，优先使用 `Wants=` 而非 `Requires=`。尽可能使用 `nofail`。

### 5. 计划维护窗口需要协调

NFS 服务器维护和服务器重启都是计划内的，但未协调。一个简单的跨团队沟通就可以预防此次事故。

**教训**：当安排了多个维护操作时，验证系统间的依赖关系是否已考虑在内。

### 6. Automount 是你的朋友

`x-systemd.automount` 不仅仅是一个便利功能——它是一个可靠性机制。通过将挂载推迟到首次访问，你将服务启动与文件系统可用性解耦。这使得系统对瞬态网络问题更具弹性。

**教训**：对所有网络文件系统使用 automount，除非有特定原因需要在启动时挂载。

### 7. 始终具备启动验证流程

一个简单的启动后健康检查脚本，验证所有关键服务是否运行，就能立即发现此次事故。没有它，这 15 分钟的中断是由一个不相关的自动健康检查才发现的。

**教训**：实施启动时验证。一个 5 行的脚本远胜于 15 分钟的中断。

## 总结

### 事件时间线

| 时间 (UTC) | 事件 |
|------------|------|
| 02:00 | 服务器开始计划中的内核更新重启 |
| 02:00:40 | 内核启动，systemd 初始化目标 |
| 02:00:42 | systemd 尝试挂载 NFS /var/log/nginx |
| 02:00:45 | NFS 挂载超时（NFS 服务器不可达） |
| 02:01:05 | var-log-nginx.mount 标记为失败 |
| 02:01:05 | remote-fs.target 因依赖失败进入失败状态 |
| 02:01:30 | Tomcat 启动，立即因依赖失败而失败 |
| 02:01:59 | Tomcat 标记为失败（结果：dependency） |
| 02:03:30 | Nginx 重试耗尽，标记为失败（结果：timeout） |
| 02:15 | 自动告警触发当班工程师 |
| 02:17 | 工程师修复 fstab，重新加载 systemd，重启服务 |
| 02:17:30 | Nginx 和 Tomcat 确认运行正常 |

面向用户服务的总停机时间：15 分钟。

### fstab 配置对比

**修复前（有问题的）：**
```
nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0
```

**修复后（已修复的）：**
```
nfs-server:/exports/logs  /var/log/nginx  nfs  _netdev,noexec,nofail,x-systemd.automount,x-systemd.device-timeout=30s  0  0
```

修复增加了四个关键选项：`_netdev` 将挂载分类为网络类型，`nofail` 防止阻塞启动，`x-systemd.automount` 将挂载推迟到首次访问，`x-systemd.device-timeout=30s` 限制了等待时间。这些选项共同确保了不可达的 NFS 服务器再也不会导致生产 Web 服务宕机。
