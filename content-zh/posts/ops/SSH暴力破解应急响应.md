---
title: "SSH 暴力破解应急响应——50 万次登录尝试如何打瘫你的堡垒机"
date: 2026-05-30
weight: 100230
slug: "ssh-brute-force-incident-response"
tags: ["security", "ssh", "linux", "incident-response", "troubleshooting"]
categories: ["Security"]
description: "SSH 暴力破解攻击事件复盘——一台使用默认配置的公网堡垒机，遭分布式暴力破解 50 万次登录尝试导致 CPU 打满，管理员无法登录，集群失联 45 分钟"
keywords: "ssh 暴力破解, fail2ban 配置, linux ssh 安全, 堡垒机防护, kubernetes 节点 ssh 访问"
draft: false
featured: true
cover:
  image: ""
  caption: "SSH 暴力破解——应急响应"
---

# SSH 暴力破解应急响应——50 万次登录尝试如何打瘫你的堡垒机

## 常见搜索词

- ssh 暴力破解防御
- fail2ban ssh 配置
- linux 服务器被 ssh 攻击
- 堡垒机安全最佳实践
- kubernetes 节点 ssh 访问安全

---

## 故障经过

**环境**: 单台堡垒机（4C/8G, Ubuntu 22.04），作为 5 个 Kubernetes 集群节点的 SSH 跳板机。公网 22 端口开放。

**时间**: 周日凌晨 03:00。Cloudflare Security Center 告警：堡垒机 SSH 端口流量异常突增。

**初始症状**: 堡垒机 CPU 飙到 100%。管理员 SSH 连接超时。系统几乎无响应。

```bash
# 管理员尝试连接时的表现
ssh admin@bastion.example.com
# 30 秒后连接超时
# 重试——同样结果
```

**影响**: 5 个 Kubernetes 集群失联 45 分钟。只能通过带外管理（iDRAC/IPMI）进行应急响应。

---

## 背景

这台堡垒机已经跑了 18 个月没出过问题。配置包含：

- 密码认证开启（"紧急情况下备用"）
- 默认 SSH 端口（22）
- 无速率限制
- 无 fail2ban
- 标准 Ubuntu 安装，默认 sshd_config

18 个月来，攻击量在逐步增加。从最初的每天几百次尝试增加到数万次。这台机器一直扛着——直到这个周日。

攻击者通过 Shodan 扫描发现了这台主机，将其加入分布式暴力破解僵尸网络。攻击高峰达到 2 小时内来自 1200+ 个独立 IP 的 50 万次登录尝试。

```bash
# 检查当前失败登录数
grep "Failed password" /var/log/auth.log | wc -l
# 最近 2 小时 51 万次失败
```

---

## 排查过程

### 第一步：评估损害

```bash
# 检查成功登录
grep "Accepted" /var/log/auth.log | grep -v "CRON" | grep -v "systemd"
# 结果：0 次非管理员的成功登录
```

幸运的是，没有攻击者成功登录。损害纯粹是可用性问题：CPU 在处理认证请求时被压垮。

### 第二步：识别攻击模式

```bash
# 攻击来源 IP 排行
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -10
```

```
48291 45.33.32.156
47321 185.220.101.42
44107 51.75.145.203
...
```

1200+ 个独立 IP，分布式协同暴力破解。

### 第三步：检查已知用户枚举

```bash
# 最常被尝试的用户名
grep "Failed password" /var/log/auth.log | awk '{print $(NF-5)}' | sort | uniq -c | sort -rn | head -10
```

```
112034 root
98431 admin
83721 ubuntu
...
```

标准凭据填充：`root`、`admin`、`ubuntu`、`test`、`deploy`、`jenkins`。

---

## 根因

1. **密码认证开启**：SSH 密钥是主要方式，但密码认证作为"备份"保留
2. **无速率限制**：每次失败尝试都消耗 CPU 进行认证处理
3. **无 fail2ban 或类似工具**：50 万次尝试后才被注意到
4. **公网 SSH 无 DDoS 防护**：堡垒机直接暴露在互联网上，没有 WAF 或 DDoS 缓解层
5. **未监控 auth.log 异常**：没有对 SSH 失败尝试异常突增设置告警

---

## 解决方案

### 紧急处置

```bash
# 1. 杀掉所有 SSH 会话释放 CPU
pkill -9 sshd
systemctl restart sshd

# 2. 临时仅允许办公室 IP 访问
iptables -A INPUT -p tcp --dport 22 -s <办公室_IP_1> -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -s <办公室_IP_2> -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

1 分钟内，CPU 从 100% 降到 5%。

### 安装配置 fail2ban

```bash
apt install fail2ban -y

cat > /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 300
ignoreip = <办公室_IP_1> <办公室_IP_2>
EOF

systemctl enable --now fail2ban
```

### 加固 SSH 配置

```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
PermitRootLogin no
MaxAuthTries 3
MaxSessions 10
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2
```

### 更换 SSH 端口（纵深防御）

```bash
# /etc/ssh/sshd_config
Port 22222

systemctl restart sshd
```

### 添加 Cloudflare Spectrum 或类似 DDoS 防护

配置 Cloudflare Spectrum 代理 SSH 流量，隐藏真实堡垒机 IP：

```
Cloudflare → Spectrum (SSH 代理) → 堡垒机 (端口 22222)
```

### 部署 CrowdSec 威胁情报

```bash
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | bash
apt install crowdsec crowdsec-firewall-bouncer-iptables -y
```

CrowdSec 与全球威胁情报网络共享攻击数据——攻击你堡垒机的 IP 会全局封禁。

### Kubernetes 集群访问审计

```bash
# 检查堡垒机上所有 kubeconfig 文件
find /home -name "kubeconfig" -o -name "*.kubeconfig" 2>/dev/null

# 轮换集群管理员凭据
kubectl config view --flatten > /tmp/admin-kubeconfig
# 撤销并重新签发所有 kubeconfig
```

### 监控告警

```bash
# SSH 失败突增告警
# Prometheus + node_exporter + alertmanager 规则：
groups:
- name: ssh-security
  rules:
  - alert: SSHFailureSpike
    expr: rate(node_ssh_failures_total[5m]) > 10
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "SSH 失败率突增（>10/秒）"
```

---

## 经验教训

- **密码认证是负担，不是备份**：SSH 密钥更安全且 CPU 消耗极低。密码认证敞开了 CPU 耗尽攻击面
- **fail2ban 是标配**：每个公网 SSH 服务器都要有 fail2ban 或同类工具。3 次重试 1 小时封禁的默认配置是最低要求
- **速率限制就是 DDoS 防护**：认证风暴是 CPU 攻击。在网络层（iptables/Cloudflare）做速率限制防止 CPU 饱和
- **审计日志需要告警**：50 万条日志不是"噪音"——是告警阈值失效的信号。如果 auth.log 突增没有传呼你，你就是盲的
- **堡垒机是关键基础设施**：一台堡垒机被攻陷或不可用会阻塞整个集群的访问通道。用和生产服务同样的安全标准对待它

---

## 总结

攻击链路：

```
攻击者通过 Shodan 扫描发现暴露的 SSH 服务器
→ 在默认 22 端口找到堡垒机
→ 发起分布式暴力破解：1200 个 IP, 50 万次尝试
→ CPU 100% 处理认证请求
→ 合法管理员被锁在外面
→ 集群 45 分钟失联
→ 紧急 IP 白名单恢复访问
→ 部署 fail2ban + SSH 加固 + Cloudflare Spectrum
```

恢复时间：45 分钟。修复部署：3 小时。这次攻击没什么技术含量——50 万次字典攻击加租来的僵尸网络 IP。但它成功了，因为服务器的配置还是 18 个月前的出厂默认设置。安全债务会累积。今天就去检查你的 SSH 配置。
