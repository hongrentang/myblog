---
title: "API Server 未授权访问事件——公网暴露的 kube-nginx 差点让集群裸奔"
date: 2026-06-03
weight: 100280
slug: "api-server-unauthorized-access"
tags: ["kubernetes", "security", "api-server", "troubleshooting"]
categories: ["Security"]
description: "Kubernetes API Server 安全事件复盘——API Server 通过公网暴露且未配置认证，匿名用户可直接查询集群资源，Shodan 扫描后 400+ 爬虫蜂拥而至"
keywords: "kubernetes api server 暴露, 匿名认证 kubernetes, kube-apiserver 安全, oidc kubernetes 认证, kubectl 未授权访问"
draft: false
featured: true
cover:
  image: ""
  caption: "API Server 未授权访问——安全事件排查"
---

# API Server 未授权访问事件——公网暴露的 kube-nginx 差点让集群裸奔

## 常见搜索词

- kubernetes api server 公网暴露
- 匿名认证 kubernetes api
- kube-apiserver 认证绕过
- 加固 kubernetes api server
- kubectl 无需认证

---

## 故障经过

**环境**: K8S v1.27, kubeadm 部署, 单控制平面节点（2C/4G），开发集群。API Server 通过 kube-nginx 反向代理暴露。

**时间**: 周五 22:00。Cloudflare Security Center 告警：`https://k8s-api.dev.example.com` 正被来自 30 个国家 400+ 个 IP 访问。

**发现过程**: 开发集群的 API Server 可从互联网访问。Shodan 扫描发现了它，自动化爬虫正在探测端点。

```bash
# 攻击者看到的内容（也是团队在日志中看到的）
curl -k https://k8s-api.dev.example.com:6443/api/v1/pods
# 返回了数据——无需认证
```

**影响**: API Server 配置了 `--anonymous-auth=true`，且匿名用户被授予了 RBAC 访问权限。3 小时内，集群资源元数据暴露在互联网上。

---

## 背景

这个开发集群本应仅限内部访问——只能通过 VPN。为了让团队访问更方便，API Server 通过 kube-nginx 反向代理暴露在公网 DNS 记录上。

Nginx 配置没有任何认证，没有 IP 白名单，只是直通到 API Server。

API Server 本身的启动参数：

```bash
kube-apiserver \
  --anonymous-auth=true \          # ← 允许匿名请求
  --authorization-mode=RBAC \
  --allow-privileged=false
```

默认情况下，Kubernetes 将 `system:anonymous` 用户绑定到 `system:discovery` ClusterRoleBinding——赋予匿名用户 API 发现端点访问权限：

```bash
# 匿名用户可以访问的内容
kubectl --server=https://k8s-api.dev.example.com:6443 --insecure-skip-tls-verify api-resources
# API 资源列表返回成功
```

在这个集群中，有人还将 `system:anonymous` 用户绑定到了 `view` ClusterRole（用于监控目的），赋予了对所有资源的读取权限：

```bash
# 有人创建了这个绑定用于"监控"
kubectl create clusterrolebinding anonymous-view \
  --clusterrole=view \
  --user=system:anonymous
```

---

## 排查过程

### 第一步：确认暴露范围

```bash
curl -sk https://k8s-api.dev.example.com:6443/api/v1/namespaces
```

API 返回了所有命名空间的列表。无需认证。

```bash
curl -sk https://k8s-api.dev.example.com:6443/api/v1/namespaces/production/secrets
# 返回 403 — view 角色不授予 Secret 访问权限
```

### 第二步：检查审计日志

审计日志显示来自 400+ IP 的 10,000+ 次匿名 API 调用。大部分是发现探测，但有几次尝试：
- 跨所有命名空间列出 Pod
- 尝试创建 Pod
- 访问 ConfigMap（包含连接字符串）
- 探测通过绑定创建进行权限提升

### 第三步：识别配置错误链

```bash
kubectl get clusterrolebinding -o json | jq -r '
  .items[] | select(.subjects[]?.name == "system:anonymous") |
  "\(.metadata.name) → \(.roleRef.name)"'
```

```
anonymous-view → view
system:discovery → system:discovery
```

`anonymous-view` 绑定是核心问题。几个月前有人为监控概念验证添加了它，从未清理。

---

## 根因

1. **API Server 暴露在互联网上**：kube-nginx 反向代理无 IP 限制、无认证、公网可访问
2. **匿名认证开启**：`--anonymous-auth=true` 允许未认证请求到达 RBAC 层
3. **匿名用户被授予 view 权限**：`anonymous-view` ClusterRoleBinding 给了 `system:anonymous` 所有资源的读取权限
4. **无网络限制**：没有 Cloudflare WAF、VPN 要求或 Nginx 级别的 IP 白名单
5. **无审计监控**：10,000+ 次匿名 API 调用未触发任何告警

---

## 解决方案

### 紧急处置

```bash
# 删除危险绑定
kubectl delete clusterrolebinding anonymous-view

# 匿名用户现在只能访问发现端点
# （默认 system:discovery ClusterRoleBinding 只授权 /api 和 /apis 的访问）
```

### 添加 API Server 认证

```yaml
# 在 kube-apiserver.yaml 中添加 OIDC 配置
spec:
  containers:
  - command:
    - kube-apiserver
    - --oidc-issuer-url=https://accounts.google.com
    - --oidc-client-id=xxxxx.apps.googleusercontent.com
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
```

### 加固反向代理

```nginx
server {
    listen 6443 ssl;
    server_name k8s-api.dev.example.com;

    # IP 白名单——仅办公室 IP
    allow 203.0.113.0/24;
    deny all;

    # 基础认证作为附加层
    auth_basic "Kubernetes API Server";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass https://127.0.0.1:6443;
    }
}
```

### 启用审计日志告警

```yaml
# Falco 规则：检测匿名用户 API 调用
- rule: Anonymous API Access
  desc: 检测来自 system:anonymous 用户的 API 请求
  condition: ka.user.name = "system:anonymous"
  output: "检测到匿名 API 访问 (user=%ka.user.name verb=%ka.verb resource=%ka.target.resource)"
  priority: WARNING
```

```yaml
# Prometheus 告警：匿名 API 突增
- alert: AnonymousAPISpike
  expr: rate(apiserver_request_anonymous_total[5m]) > 10
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "匿名 API 请求率异常偏高"
```

---

## 经验教训

- **API Server 永远不应直接暴露在互联网上**：始终使用 VPN、Cloudflare Access 或至少 IP 白名单。kube-nginx 代理不是安全边界
- **`system:anonymous` 是真实用户**：像对待任何其他用户一样对待匿名访问。审计匿名用户拥有哪些权限，收回发现端点以外的所有权限
- **默认匿名权限是安全的——自定义绑定是危险的**：默认 `system:discovery` 绑定没问题。将 `system:anonymous` 添加到 `view`、`edit` 或 `admin` 相当于让你的集群公开可读
- **OIDC 是 API 认证的标准**：超越静态 Token 和客户端证书。OIDC 提供短生命周期的 Token、组映射和与现有 SSO 的集成
- **VPC/仅内部不应是唯一的防线**："只能通过 VPN 访问"不是安全策略——是侥幸心理。纵深防御同样适用于网络安全

---

## 总结

攻击链路：

```
开发集群 API Server 通过 kube-nginx 在公网 DNS 上暴露
→ Shodan 扫描互联网中的开放 Kubernetes API 端点
→ 发现 k8s-api.dev.example.com:6443
→ curl /api/v1/pods 返回数据——无需认证
→ 3 小时内 400+ 自动化爬虫探测该端点
→ system:anonymous 用户拥有 view ClusterRoleBinding
→ 包含连接字符串的 ConfigMap 被访问
→ 匿名访问未触发任何告警
```

发现：Cloudflare 异常流量告警。修复：1 小时完成加固。根因修复：删除 anonymous-view 绑定 + IP 白名单 + OIDC。API Server 是皇冠上的明珠——别把门开着让人随便进。
