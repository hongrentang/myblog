---
title: "TLS 证书过期排查实录——从 NET::ERR_CERT_DATE_INVALID 到证书全生命周期管理"
date: 2026-05-25
weight: 100170
slug: "tls-certificate-expired-troubleshooting"
tags: ["tls", "certificate", "security", "ingress", "troubleshooting"]
categories: ["安全"]
description: "网站突然报 NET::ERR_CERT_DATE_INVALID，排查从怀疑 DNS、怀疑代理到最终定位证书过期，以及如何用 cert-manager 自动化轮转的完整过程"
keywords: "tls certificate expired, ssl error, cert-manager, letsencrypt, certificate renewal"
draft: false
featured: true
cover:
  image: "/images/tls-cert-expired-banner.svg"
  caption: "TLS 证书过期排查"
---

# TLS 证书过期排查实录——从 NET::ERR_CERT_DATE_INVALID 到证书全生命周期管理

## 问题现象

周一早上 10 点，用户反馈官网打不开。浏览器显示：

```
NET::ERR_CERT_DATE_INVALID
```

用 curl 试一下：

```bash
curl -v https://www.example.com
```

```
*   Trying 203.0.113.10:443...
* Connected to www.example.com (203.0.113.10) port 443 (#0)
* ALPN: offers h2,http/1.1
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* Certificate chain
  *  subject: CN=www.example.com
  *  start date: May 20 2025 00:00:00 GMT
  *  expire date: May 20 2026 00:00:00 GMT
  *  subjectAltName: host "www.example.com" matched
* SSL certificate verify result: 10 (certificate has expired)
* Closing connection
* TLS error: certificate has expired
curl: (60) SSL certificate problem: certificate has expired
```

`certificate has expired`——证书已经过期了。

环境的访问链路：

```
用户 → Cloudflare → Nginx Ingress Controller → 后端 Service
```

TLS 证书挂在 Nginx Ingress 上，证书是 Let's Encrypt 签发的一年期证书。

**影响**：所有 HTTPS 访问全部中断。用户看到"您的连接不是私密连接"的安全警告。API 客户端直接报错无法请求。

## 排查过程

### 错误尝试 1：怀疑 DNS / CDN 问题

浏览器报安全错误，第一反应——是不是 Cloudflare 的回源配置出了问题？或者 DNS 解析到了错误的 IP？

```bash
dig +short www.example.com
```

```
203.0.113.10
```

IP 是对的。

```bash
curl -I http://www.example.com
```

```
HTTP/1.1 301 Moved Permanently
Location: https://www.example.com/
```

HTTP 能正常返回 301 重定向——说明源站是活的，问题只在 HTTPS 层面。

**踩坑点**：HTTP 正常、HTTPS 报错——这个问题 90% 出在证书层，不是 DNS 也不是网络。浏览器对 HTTP 和 HTTPS 走的是两套验证流程，HTTPS 多了一层证书校验。HTTP 能通就排除了网络和 DNS 的问题。

### 错误尝试 2：怀疑 Cloudflare 回源证书配置

"可能是 Cloudflare 的 Edge Certificate 过期了？"——登录 Cloudflare 控制台看了眼：

```
Edge Certificates:
  www.example.com          Valid         2025-05-20 ~ 2026-05-20
  *.example.com            Valid         2025-05-20 ~ 2026-05-20
```

Cloudflare 侧的证书是正常的。问题在回源证书——Cloudflare 到 Nginx Ingress 这一段。

Cloudflare 的配置是 **Full (strict)** 模式，意味着 Cloudflare 到源站也要做 TLS 验证。

```bash
# 绕过 CDN，直接请求源站
curl -v https://203.0.113.10 --resolve www.example.com:443:203.0.113.10
```

```
* SSL certificate verify result: 10 (certificate has expired)
```

直接请求源站也报过期——确认了是源站的证书问题，不是 Cloudflare。

### 错误尝试 3：重启 Ingress Controller

"重启看看会不会重新加载证书"：

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

等 Pod 滚动更新完：

```bash
kubectl get pods -n ingress-nginx
```

```
NAME                                       READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7d9f8c6b8f-x1     1/1     Running   0          2m
```

再访问——依然 `NET::ERR_CERT_DATE_INVALID`。

**踩坑点**：重启 Nginx 不会自动续期证书。证书是直接写在 Secret 里的静态资源（通过 kubectl apply 或者 cert-manager 写入），重启 Pod 只是重新加载 Secret，但 Secret 里的证书本身已经过期了，重新加载还是一样过期。这就好比你有一张过期的身份证，换了个钱包装着——身份证还是过期的。

### 真正的发现：查看 Ingress 证书

检查 Ingress 上挂的证书：

```bash
kubectl describe ingress www-example-com -n production
```

```
Annotations:          
  cert-manager.io/cluster-issuer: letsencrypt-prod
TLS:
  hosts:
    - www.example.com
  secretName: www-example-com-tls
```

找到证书 Secret：

```bash
kubectl get secret www-example-com-tls -n production -o yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: www-example-com-tls
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: (base64)
  tls.key: (base64)
```

解码查看证书内容：

```bash
kubectl get secret www-example-com-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep -E "Not Before|Not After"
```

```
Not Before: May 20 00:00:00 2025 GMT
Not After : May 20 00:00:00 2026 GMT
```

```
# 看有效期还剩多久
kubectl get secret www-example-com-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -enddate -noout -checkend 0
```

```
Certificate will expire
```

```bash
# 具体还有几天过期
kubectl get secret www-example-com-tls -n production -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -enddate -noout -dates
```

```
notAfter=May 20 00:00:00 2026 GMT
```

今天是 2026 年 5 月 25 日——已经过期 5 天了。

检查 cert-manager 的状态：

```bash
kubectl get certificate -n production
```

```
NAME                  READY   SECRET                AGE
www-example-com-tls   False   www-example-com-tls   365d
```

```bash
kubectl describe certificate www-example-com-tls -n production
```

```
Status:
  Conditions:
    Reason:              Expired
    Status:              True
    Type:                Expired
  Not After:             2026-05-20T00:00:00Z
  Renewal Time:          2026-04-20T00:00:00Z
  ...
Events:
  Warning  RenewalError  30d   cert-manager  failed to renew: acme: error: 403 :: POST :: Invalid order
```

**RenewalError**——30 天前 cert-manager 尝试续期时失败了，原因 ACME 订单无效（403）。之后 cert-manager 一直在重试但都没成功。

## 根因分析

| 层面 | 原因 |
|------|------|
| 直接原因 | Let's Encrypt 证书于 2026-05-20 到期，所有 HTTPS 服务中断 |
| 续期失败 | cert-manager 在 2026-04-20 尝试续期时 ACME 返回 403（域名验证失败/API 限频） |
| 监控缺失 | 没有证书过期告警，也没有提前通知 |
| 为什么没自动恢复  | cert-manager 重试策略配置不足，30 天内持续失败但未被发现 |

证书续期的完整流程应该是：

```
证书到期前 30 天
  → cert-manager 发起 ACME 验证
  → 验证通过后签发新证书
  → cert-manager 自动更新 Secret
  → Nginx Ingress 自动加载新证书
```

但这次续期在 ACME 验证阶段卡住了——cert-manager 申请续期时 Let's Encrypt API 返回了 403。常见原因：

1. **域名所有权验证失败**——HTTP-01 挑战时 Let's Encrypt 无法访问验证文件
2. **API 限频**——同一域名短期内请求次数过多
3. **ACME 客户端版本过旧**——cert-manager 版本太老，Let's Encrypt API 更新后不兼容
4. **DNS-01 挑战的凭据过期**——用于修改 DNS TXT 记录的 API Token/Secret 过期了

这次的原因是 DNS-01 挑战的 Cloudflare API Token 过期了。

```bash
# 查看 cert-manager 的 ClusterIssuer 配置
kubectl describe clusterissuer letsencrypt-prod
```

```
Spec:
  Acme:
    Solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

```bash
# 检查 API Token Secret 是否过期
kubectl get secret cloudflare-api-token -n cert-manager -o yaml
```

Secret 里的 API Token 有效期只有 6 个月——恰好在上次签发后 6 个月过期。cert-manager 续期时拿着过期的 Token 去 Let's Encrypt 做 DNS-01 挑战，自然被 Cloudflare API 拒绝了。

## 解决方案

### 紧急恢复：手动申请新证书

先让网站恢复再说：

```bash
# 方案一：用 certbot 手动申请（最快）
certbot certonly --manual --preferred-challenges dns \
  -d www.example.com -d *.example.com
```

按照提示在 DNS 添加 TXT 记录，验证通过后拿到新证书：

```bash
# 证书已签发
ls -la /etc/letsencrypt/live/www.example.com/
```

```bash
# 创建 K8S Secret
kubectl create secret tls www-example-com-tls \
  --cert=/etc/letsencrypt/live/www.example.com/fullchain.pem \
  --key=/etc/letsencrypt/live/www.example.com/privkey.pem \
  -n production --dry-run=client -o yaml | kubectl apply -f -
```

```bash
# 验证
curl -I https://www.example.com
# 200 OK
# SSL certificate verify ok
```

**踩坑点**：手动申请的 Let's Encrypt 证书有效期还是 90 天，且 certbot 不会在 K8S 环境自动续期。这是临时方案，不是长期方案。

### 根本修复：更新 cert-manager 的 Cloudflare Token 并触发续期

```bash
# 1. 更新 Cloudflare API Token（在 Cloudflare 后台生成新的）
# 权限：Zone:DNS:Edit

# 2. 更新 K8S Secret
kubectl delete secret cloudflare-api-token -n cert-manager
kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=<新Token> \
  -n cert-manager

# 3. 删除旧证书（让 cert-manager 重新签发）
kubectl delete certificate www-example-com-tls -n production

# 4. 重新创建 Certificate 资源
cat <<'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: www-example-com-tls
  namespace: production
spec:
  secretName: www-example-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - www.example.com
    - example.com
  renewBefore: 720h  # 提前 30 天续期
EOF
```

```bash
# 5. 查看续期状态
kubectl get certificate -n production -w
```

```
NAME                  READY   SECRET                AGE
www-example-com-tls   True    www-example-com-tls   1m
```

```bash
# 6. 验证新证书有效期
kubectl get secret www-example-com-tls -n production \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -text -noout | grep -E "Not Before|Not After"
```

```
Not Before: May 25 00:00:00 2026 GMT
Not After : Aug 23 00:00:00 2026 GMT
```

新证书有效期 90 天。

### 验证恢复

```bash
# 1. 浏览器访问
# https://www.example.com — 正常显示，无安全警告

# 2. curl 验证
curl -vI https://www.example.com 2>&1 | grep "SSL certificate"
# SSL certificate verify ok.

# 3. 检查 Nginx Ingress 证书加载
kubectl exec -n ingress-nginx ingress-nginx-controller-xxx -- \
  openssl s_client -connect localhost:443 -servername www.example.com \
  < /dev/null 2>/dev/null | openssl x509 -noout -dates

# 4. 检查 cert-manager 状态
kubectl get certificate -n production
# READY: True
```

✅ **恢复确认**：
- 浏览器 HTTPS 正常访问
- curl SSL verify ok
- 新证书有效期从当天开始
- cert-manager 状态 READY

### 长期预防——证书全生命周期管理

```bash
# 1. 证书过期告警（提前 30 天 + 7 天 + 1 天）
# Prometheus Blackbox Exporter
# probe_ssl_earliest_cert_expiry < (30 * 86400)

# 或者用 cert-manager 自带的事件
# 监听 Certificate 的 RenewalError 事件

# 2. cert-manager 续期配置优化
# cert-manager.yaml
spec:
  renewBefore: 720h  # 提前 30 天
  # 续期失败后重试间隔（默认是 5m，可以根据需要延长）
  # 增加监测频率

# 3. Cloudflare API Token 到期前提醒
# Token 设置成 1 年有效期，日历提醒
# 或者使用 cert-manager 的外部 Secret 存储（Vault、AWS Secret Manager）

# 4. 证书到期前压测续期流程
# 每季度手动演练一次证书续期
# 测试 DNS-01 / HTTP-01 两种挑战方式

# 5. 监控证书有效期
# 脚本定期检查
for domain in www.example.com api.example.com; do
  expiry=$(openssl s_client -connect $domain:443 -servername $domain </dev/null 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
  expiry_epoch=$(date -d "$expiry" +%s)
  now_epoch=$(date +%s)
  days_left=$(( (expiry_epoch - now_epoch) / 86400 ))
  echo "$domain: $days_left days remaining"
  if [ $days_left -lt 7 ]; then
    echo "WARNING: Certificate for $domain expires in $days_left days!"
  fi
done
```

## 教训总结

1. **证书过期不是"会不会发生"的问题，是"什么时候发生"的问题。** Let's Encrypt 的 90 天有效期意味着每年要续期 4 次。每次续期都可能失败（API 限频、Token 过期、域名验证失败）。自动化的前提是**每个环节都要有监控**——cert-manager 的续期尝试、ACME 验证结果、证书的 NotAfter 时间。

2. **浏览器报 NET::ERR_CERT_DATE_INVALID 要先看证书有效期。** 不要先重启 Ingress、查 DNS、问 CDN 客服。一个简单的 `openssl s_client` 命令就能直接看到证书的起止时间、签发人和 Subject。排查 SSL 问题最忌讳在非证书层瞎转。

3. **cert-manager 的 RenewalError 不是永远会自动恢复的。** 如果底层凭据（Cloudflare API Token、AWS Route53 key）过期了，cert-manager 重试一万次也续不了。这个 Token 本身也需要生命周期管理。建议把 cert-manager 用的外部凭据也纳入监控，在过期前通知。

4. **TLS 是安全的第一道门，但也是最容易被忽略的门。** 服务挂了有 CPU 告警、磁盘告警、内存告警，但证书过期经常没人管。监控系统一定要加证书有效期指标——用 Blackbox Exporter 的 `probe_ssl_earliest_cert_expiry` 或者直接用脚本扫。证书过期是整个 HTTPS 基础设施的"单点故障"，而且一过期就是全部端口一起挂。
