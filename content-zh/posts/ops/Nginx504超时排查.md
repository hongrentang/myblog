---
title: "Nginx 504 Gateway Timeout 排查——从 upstream 超时到全链路优化"
date: 2026-05-20
weight: 100090
slug: "nginx-504-gateway-timeout-troubleshooting"
tags: ["nginx", "network", "troubleshooting"]
categories: ["网络"]
description: "Nginx 504 Gateway Timeout 的完整排查思路，从误判 upstream 到定位连接耗尽和 keepalive 配置问题"
keywords: "nginx 504, gateway timeout, upstream timed out, nginx调优, proxy_read_timeout"
draft: false
featured: true
cover:
  image: "/images/nginx-504-banner.svg"
  caption: "Nginx 504 Gateway Timeout 排查"
---

# Nginx 504 Gateway Timeout 排查——从 upstream 超时到全链路优化

## 问题现象

线上 API 开始随机返回 504。不是全部请求都挂，是间歇性的——有些请求正常返回，有些等了几十秒后吐一个 504。

```bash
curl -I https://api.example.com/v1/orders
# 等了 30 秒后返回：
# HTTP/2 504
```

浏览器里 F12 看 Network 面板，`waiting (TTFB)` 长达 20-30 秒，然后直接 504。后端应用日志显示请求正常处理完了（耗时不到 200ms），但 Nginx 没把响应拿回去。

从 Nginx 视角看：

```bash
tail -f /var/log/nginx/access.log | grep " 504 "
```

```
10.0.0.1 - - [20/May/2026:14:30:15 +0800] "GET /v1/orders" 504 0 "-" "curl/7.68" 0.050 30.121
```

`upstream_response_time` 30 秒，`request_time` 也是 30 秒——Nginx 等 upstream 等了 30 秒，然后超时返回 504。

**影响**：核心 API 间歇性不可用，前端大量报错，用户无法下单。

## 排查过程

### 错误尝试 1：以为 upstream 真的慢

504 的直接意思是"upstream 没在时间内响应"。第一反应——查后端应用。

```bash
# 直接请求后端应用的端口（绕过 Nginx）
curl -w "time_total: %{time_total}s\n" http://127.0.0.1:8080/v1/orders
```

```
<正常返回>
time_total: 0.085s
```

85ms 就返回了？那 Nginx 为什么报 30 秒超时？

再试几次：

```bash
for i in $(seq 1 10); do
    curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" http://127.0.0.1:8080/v1/orders
done
```

```
200 0.082s
200 0.091s
200 0.078s
200 0.095s
200 0.085s
200 0.088s
200 0.081s
200 0.093s
200 0.079s
200 0.086s
```

全部正常，全部在 100ms 内。后端没问题。

### 错误尝试 2：调大 proxy_read_timeout

既然 upstream 不慢，那可能是 Nginx 的超时配置太短了。

```bash
# 检查当前超时配置
grep -r "proxy_read_timeout" /etc/nginx/
```

```
proxy_read_timeout 30s;
```

30 秒，确实跟 504 表现一致。改成 60 秒：

```bash
# 编辑配置
sed -i 's/proxy_read_timeout 30s;/proxy_read_timeout 60s;/' /etc/nginx/conf.d/default.conf
nginx -s reload
```

再测：

```bash
curl -I https://api.example.com/v1/orders
```

等了 60 秒，依然 504。只是从 30 秒超时变成了 60 秒超时——超时不代表等待时间不够，而是连接本身就有问题。

**踩坑点**：调大超时不是解决方案。如果 upstream 正常（100ms 返回），30 秒超时绰绰有余。调成 60 秒只会让用户等更久，不会降低 504 的**频率**。需要搞清楚的是：为什么 Nginx 和 upstream 之间有的请求正常、有的请求卡死。

### 真正的发现：连接复用问题

直接请求后端（绕过 Nginx）一直正常，但通过 Nginx 就间歇性 504。那问题一定出在 Nginx 和 upstream 的**连接**上。

检查 Nginx 的 upstream 配置：

```bash
cat /etc/nginx/conf.d/default.conf
```

```
upstream backend {
    server 127.0.0.1:8080;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

看起来正常？注意 `proxy_http_version 1.1` 配合 `Connection ""`——这是为了让 Nginx 复用和后端的 keepalive 连接。但是：

```bash
# 检查有没有 keepalive 指令
grep -r "keepalive" /etc/nginx/
```

没有 `keepalive` 指令。

```bash
# 看下默认的 upstream 配置
cat /etc/nginx/nginx.conf | grep -A 10 "upstream"
```

```
# 没有 keepalive 相关配置
```

**这就是问题所在**：Nginx 和 upstream 之间开启了 HTTP/1.1 keepalive（通过 proxy_http_version 1.1 和空的 Connection header），但没有配置 upstream keepalive 连接池。Nginx 对每个 upstream 默认只会保持 1 个空闲 keepalive 连接，当并发请求超过这个数量时，新的请求需要建立新连接。

但更关键的是——Nginx 的 upstream 连接池默认最大连接数是 `worker_connections`，而 proxy_keepalive 没配的情况下，连接会在请求结束后关闭。高并发下，大量连接处于 TIME_WAIT 状态，新的连接请求被阻塞。

```bash
# 确认 TIME_WAIT 状态
ss -s
```

```
Total: 12345 (kernel)
TCP:   9876 (estab 234, closed 6543, timewait 3210, ...)
```

3210 个 TIME_WAIT。

```bash
# 看 Nginx 和后端之间的连接状态
ss -tan | grep 8080 | awk '{print $1}' | sort | uniq -c
```

```
   8 ESTAB
  35 TIME-WAIT
```

35 个 TIME_WAIT 连接到后端。这些连接占用了端口资源，新连接需要等到 TIME_WAIT 释放。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | Nginx 和后端之间的连接池不足，高并发下大量 TIME_WAIT 堆积，新连接被阻塞 |
| 配置缺陷 | upstream 块没有配置 `keepalive`，Nginx 无法复用和后端的连接 |
| 触发条件 | 突发流量导致并发连接数激增 |

一句话：**Nginx 开启了 HTTP/1.1 keepalive（为了支持长连接），但没配 `keepalive` 指令来限制空闲连接池大小。高并发下旧连接来不及复用就关闭了，大量 TIME_WAIT 堆积导致连接耗尽。**

## 解决方案

### 方案 A：快速恢复——扩大 worker_connections（临时止血）

```bash
# 编辑 nginx.conf
vim /etc/nginx/nginx.conf
```

```
worker_connections 4096;
# 原来可能是 1024
```

```bash
nginx -s reload
```

这能暂时缓解连接不足的问题，但不是根本解决。

### 方案 B：配置 upstream keepalive（推荐，根治）

```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 128;  # 保持 128 个空闲 keepalive 连接
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # 添加超时配置（合理值，不是盲目加大）
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 10s;
    }
}
```

```bash
# 验证配置语法
nginx -t

# 重载
nginx -s reload
```

**keepalive 为什么有效**：

| 配置 | 作用 |
|------|------|
| `keepalive 128` | 保持 128 个空闲连接在后端，请求直接复用，避免反复建连 |
| `proxy_http_version 1.1` + `Connection ""` | 告诉后端复用 HTTP/1.1 长连接 |

```bash
# 配置后验证 TIME_WAIT 是否下降
ss -tan | grep 8080 | awk '{print $1}' | sort | uniq -c
```

```
  42 ESTAB
   3 TIME-WAIT
```

TIME_WAIT 从 35 降到 3。

### 方案 C：启用连接复用 + 调优内核参数（生产环境）

```bash
# 内核参数调优——减少 TIME_WAIT 堆积
cat >> /etc/sysctl.conf << 'EOF'
# 允许 TIME_WAIT 连接端口被快速回收
net.ipv4.tcp_tw_reuse = 1
# 增大端口范围，避免端口耗尽
net.ipv4.ip_local_port_range = 1024 65535
# 减小 TIME_WAIT 超时时间（默认 60s）
net.ipv4.tcp_fin_timeout = 15
EOF

sysctl -p
```

**三种方案对比**：

| 方案 | 适用场景 | 效果 | 永久？ |
|------|----------|------|--------|
| A. 扩大 worker_connections | 紧急止血 | 暂时缓解 | ❌ 治标 |
| B. 配置 keepalive | 普遍适用 | 显著减少建连开销 | ✅ 治本 |
| C. 内核参数调优 | 高并发生产 | 降低 TIME_WAIT 影响 | ✅ 辅助 |

### 验证恢复

```bash
# 1. 确认 504 消失
tail -f /var/log/nginx/access.log | grep -v " 504 "

# 2. 验证连接复用
curl -I https://api.example.com/v1/orders
# 应该正常返回 200

# 3. 检查 Nginx upstream 状态
curl http://127.0.0.1/nginx_status
# 或者
cat /var/log/nginx/stub_status
# Active connections: 42
# Writing: 2 Waiting: 3

# 4. 观察 TIME_WAIT
watch -n 2 'ss -tan | grep 8080 | awk "{print \$1}" | sort | uniq -c'
```

✅ **恢复验证**：
- curl 不再返回 504
- TIME_WAIT 连接数明显下降（从 35 降到个位数）
- 前端页面恢复正常

## 长期预防

```bash
# 1. 监控 Nginx upstream 状态
# Prometheus: nginx_upstream_response_time_seconds > 10
# 发现响应时间超过 10 秒就告警，不用等到 504

# 2. 定期检查 TIME_WAIT 数量
# 阈值：TIME_WAIT > 1000 触发告警

# 3. 压测验收
# 新服务上线前用 wrk 做压力测试
wrk -t4 -c200 -d30s https://api.example.com/v1/orders
# 确认没有 50x 错误
```

## 教训总结

1. **504 不一定是 upstream 真的慢**。直接请求后端确认响应时间，再决定是调大超时还是排查连接问题。盲目调大 proxy_read_timeout 只会让用户等更久，不会降低 504 频率。

2. **Nginx upstream keepalive 是个容易忽略的配置**。开了 HTTP/1.1 但没配 `keepalive` 指令，连接还是在不断重建。这不是 bug，但效果跟没开 keepalive 差不多。

3. **TIME_WAIT 不会自己消失**。高并发场景下，不配 keepalive + 不调内核参数，TIME_WAIT 会持续堆积直到端口耗尽。`ss -s` 和 `ss -tan | uniq -c` 是排查这类问题的第一组命令。

4. **验证要覆盖正常路径和异常路径**。直接 curl 后端绕过 Nginx 能快速定位问题出在代理层还是应用层。这个简单的对比操作能省掉大量排查时间。
