---
title: "Nginx 从入门到精通 · 第5篇：HTTPS 配置"
date: 2025-01-09
weight: 1105
draft: false
tags: ["nginx"]
---

## 为什么需要 HTTPS？

- **加密**：防止中间人窃听和篡改
- **身份验证**：确认你访问的网站是真实的
- **SEO**：Google 对 HTTPS 网站有排名加分
- **浏览器标记**：HTTP 网站会被标记为"不安全"
- **合规要求**：支付、隐私数据必须使用 HTTPS

---

## SSL/TLS 证书获取

### Let's Encrypt（免费推荐）

使用 Certbot 自动获取和续期：

```bash
# 安装 Certbot
sudo apt install certbot python3-certbot-nginx -y

# 获取并自动配置 Nginx
sudo certbot --nginx -d example.com -d www.example.com

# 仅获取证书（手动配置）
sudo certbot certonly --nginx -d example.com

# 查看证书
sudo certbot certificates

# 测试续期
sudo certbot renew --dry-run
```

### 配置自动续期

```bash
# 添加 cron 任务，每天检查两次
echo "0 */12 * * * /usr/bin/certbot renew --quiet" | sudo tee -a /etc/crontab
```

Certbot 默认自动添加了 systemd timer，多数情况下无需手动添加 cron。

---

## 基础 HTTPS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/example;
    index index.html;

    # 其他配置...
}

# HTTP 自动跳转到 HTTPS
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

---

## SSL 配置优化

### 协议版本

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
```

**不要**包含 TLSv1.0 和 TLSv1.1，它们已被废弃且有安全漏洞。TLSv1.3 是当前最新最安全的协议。

### 加密套件

```nginx
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

ssl_prefer_server_ciphers on;
```

推荐使用现代浏览器的通用配置，可以在 [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) 在线生成。

### DH 参数

```nginx
ssl_dhparam /etc/nginx/dhparam.pem;
```

生成 DH 参数（需要几分钟）：

```bash
sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
```

> 2048 位在安全性和性能之间取得了平衡，4096 位更安全但生成和握手都更慢。

### HSTS

```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

告诉浏览器：未来一年内，该域名**只允许**通过 HTTPS 访问。

**注意**：启用 HSTS 前务必确认 HTTPS 已经完全正常工作，否则会把网站锁在浏览器外面。

### 完整 HTTPS 优化配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    # 证书
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_dhparam         /etc/nginx/dhparam.pem;

    # 协议和加密
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;

    # 会话缓存（提升性能）
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    root /var/www/example;
    index index.html;
}
```

---

## OCSP Stapling

OCSP（在线证书状态协议）用于浏览器检查证书是否被吊销。

没有 OCSP Stapling 时，浏览器需要额外请求 CA 的 OCSP 服务器，导致握手变慢。

OCSP Stapling 让 Nginx 自己缓存证书状态，在 TLS 握手时直接返回给浏览器：

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
```

---

## 多域名 HTTPS

使用 SNI（Server Name Indication）在一个 IP 上托管多个 HTTPS 站点：

```nginx
server {
    listen 443 ssl http2;
    server_name site1.com;
    ssl_certificate     /etc/letsencrypt/live/site1.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site1.com/privkey.pem;
}

server {
    listen 443 ssl http2;
    server_name site2.com;
    ssl_certificate     /etc/letsencrypt/live/site2.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/site2.com/privkey.pem;
}
```

> 老旧的 Windows XP/IE 不支持 SNI，会默认匹配第一个 server 块。如果还在支持这些客户端，可以把这个 server 块设为默认的通用证书。

---

## 安全加固检查

### 使用测试工具

部署 HTTPS 后，一定要做安全测试：

**在线测试**：[https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)

**命令行测试**：

```bash
# 查看证书信息
openssl s_client -connect example.com:443 -servername example.com

# 测试协议支持
nmap --script ssl-enum-ciphers -p 443 example.com
```

### 评级目标

Qualys SSL Labs 评分 ≥ **A** 是基准要求，目标 **A+**。

### 常见安全头

```nginx
# 不推荐！一个全功能的配置示例：
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "0" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

> ⚠️ 注意：上述部分头部在现代浏览器中已经过时或被默认行为替代。X-XSS-Protection 设为 `0` 是因为现代浏览器（Chrome）已经移除了内置的 XSS 过滤器，设为 `1; mode=block` 反而可能在旧浏览器带来风险。推荐使用 Content-Security-Policy（CSP）作为替代方案。

---

## 性能优化

### Session 复用

SSL/TLS 握手很慢（尤其是 RSA 密钥交换时），启用会话复用可显著减少握手次数：

```nginx
ssl_session_cache shared:SSL:10m;    # 共享缓存 10MB，约能存 40000 个会话
ssl_session_timeout 1d;              # 会话缓存 1 天
ssl_session_tickets off;             # 关闭 session ticket（某些场景下可开启）
```

### HTTP/2

```nginx
listen 443 ssl http2;
```

只需加上 `http2` 参数即可启用 HTTP/2。效果：

- 多路复用：一个连接并行传输多个资源
- 头部压缩：HPACK 减少头部开销
- Server Push（已不推荐使用）：服务端主动推送资源

---

## 常见问题

### 证书过期

```bash
# 查看证书到期时间
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -dates

# 查看还有几天到期
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -enddate | cut -d= -f2 | xargs -I{} date -d "{}" +%s | xargs -I{} sh -c 'echo $((({} - $(date +%s)) / 86400))'
```

Let's Encrypt 证书有效期为 90 天，务必确保自动续期正常。

### 证书链不完整

某些 Android 设备可能不包含中间证书，需要配置完整的证书链：

```nginx
# fullchain.pem 已包含完整链（推荐）
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;

# 或者手动拼接
ssl_certificate /path/to/cert.pem;
ssl_trusted_certificate /path/to/chain.pem;
```

### 混合内容

页面通过 HTTPS 加载，但引用了 HTTP 资源（图片、脚本），浏览器会阻止加载。

**检查方法**：浏览器开发者工具 → Console 中会报 `Mixed Content` 错误。

**修复**：确保所有资源使用 HTTPS 或协议相对 URL `//example.com/image.png`。

---

## 小结

HTTPS 已经成为网站标配。这一篇我们从证书获取、基础配置、安全优化到性能调优，覆盖了 HTTPS 部署的全流程。

---

