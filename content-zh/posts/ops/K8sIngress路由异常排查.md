---
title: "K8s Ingress 路由异常排查——流量被错误转发到旧服务的 2 小时故障纪实"
date: 2026-06-09
weight: 100450
slug: "ingress-routing-failure"
tags: ["kubernetes", "networking", "troubleshooting", "ingress", "nginx"]
categories: ["Troubleshooting"]
description: "一次 Kubernetes Ingress 路由异常故障分析——错误的 rewrite-target 注解与路径正则冲突导致生产流量被转发到错误的后端服务，造成数据污染与 2 小时中断"
keywords: "kubernetes ingress 排查, nginx ingress 路径重写错误, ingress 路由到错误服务, k8s ingress 404, ingress 配置调试"
draft: false
featured: true
cover:
  image: ""
  caption: "K8s Ingress 路由异常排查实录"
---

# K8s Ingress 路由异常排查——流量被错误转发到旧服务的 2 小时故障纪实

## 常见搜索词

如果你是通过搜索找到这里，这篇文章覆盖以下场景：

- kubernetes ingress 路由到错误的服务 排查
- nginx ingress rewrite-target 注解配置错误
- kubernetes ingress 新部署后返回 404
- ingress 路径正则冲突 多条规则重叠
- nginx ingress 部分流量被转发到旧后端
- kubectl ingress 更新后路由异常
- ingress-nginx 配置验证

---

## 故障经过

**环境**：K8S v1.28，nginx-ingress-controller v1.8，3 个微服务——orders-v1、orders-v2、users。Ingress-NGINX 通过 Helm 安装在 `kube-system` 命名空间。

**时间**：周二 14:00，计划内部署窗口。

**症状**：监控面板显示 `/api/v2/orders` 端点出现 404 错误激增。初看 orders-v2 服务似乎健康——Pod 在运行，存活探测通过，指标显示内部处理正常。但真相更隐蔽：部分发往 `/api/v2/orders` 的请求被**静默路由**到旧的 orders-v1 后端，而不是新的 orders-v2。数据库中开始出现数据不一致，订单被拆分在两个版本的服务中处理，部分成功率徘徊在 60% 左右。

```bash
# 触发的告警
Service: orders-v2  |  Error: GET /api/v2/orders/123 → 404 Not Found  (ingress 返回)
Service: orders-v2  |  Error: POST /api/v2/orders → 201 Created 但请求体被 orders-v1 处理（错误 schema）
Database: order-svc  |  Warning: 38% 的订单缺少 v2 schema 中预期的字段
```

**影响**：约 40% 的订单流量降级。结账流程部分失败，产生包含缺失字段的损坏订单记录。部分订单由 orders-v1 创建但后续被 orders-v2 处理，引发下游服务的级联故障。14:05 确认为影响收入的 P0 级事故，完整恢复（包括数据库修复）耗时约 2 小时。

---

## 背景

### nginx-ingress-controller 的工作原理

NGINX Ingress Controller 将 Kubernetes Ingress 资源翻译为 NGINX 配置文件（`nginx.conf`）并在配置变更时重新加载 NGINX。数据流如下：

```
Ingress 资源 (YAML) → Ingress Controller (watch 循环) → nginx.conf 生成 → NGINX 重新加载 → 流量路由
```

当你创建或更新 Ingress 资源时，控制器会：
1. 通过 Kubernetes API Watch 机制检测变更
2. 重新生成 NGINX 配置模板
3. 使用 `nginx -t` 验证生成的配置
4. 验证通过后重新加载 NGINX

关键在于：第 3 步只检查**语法错误**，不检查语义正确性。一个逻辑错误但语法正确的配置会成功加载。

### 路径匹配类型

nginx-ingress 支持三种路径类型：

| 类型 | 行为 | 示例 |
|---|---|---|
| `Exact` | 精确匹配路径（`/api/v2/orders` 只匹配该路径） | 无前缀匹配 |
| `Prefix` | 匹配以给定前缀开头的路径（`/api` 匹配 `/api/`、`/api/v1/` 等） | `/api` 前缀 |
| `ImplementationSpecific` | 由 NGINX 特定行为决定——路径被视为 NGINX location 模式（可包含正则） | `/api/v2/orders(/\|$)(.*)` |

`ImplementationSpecific` 类型配合正则表达式是复杂性——以及 Bug——的根源。

### rewrite-target 注解

`nginx.ingress.kubernetes.io/rewrite-target` 控制请求发送到后端服务前的路径重写方式。路径模式中的捕获组可以作为 `$1`、`$2` 等引用：

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
```

如果 Ingress 路径是 `/api/v2/orders(/|$)(.*)` 而 rewrite-target 是 `/$2`，则：
- 请求：`/api/v2/orders/123` → 捕获 `$2 = "orders/123"` → 重写为：`/orders/123`
- 请求：`/api/v2/orders` → 捕获 `$2 = ""` → 重写为：`/`

这个机制非常强大，但在配置间复制粘贴时也极其容易出错。

### Ingress Controller 重载机制

nginx-ingress controller 会在 `sync-period`（默认 30 秒）内批量处理配置变更。当 Ingress 资源发生变化时：

1. 控制器通过 informer watch 检测到变更
2. 等待同步周期以批量处理更多变更
3. 重新生成 NGINX 模板
4. 执行 `nginx -t` 验证语法
5. 向 NGINX 发送重载信号（SIGHUP）

整个过程是异步的。没有内置验证机制来检查路径冲突、路由重叠或注解正确性。

---

## 排查过程

### 第 1 步：检查 Ingress 资源

第一步——列出所有 Ingress 资源以了解路由全景：

```bash
kubectl get ingress -A
```

```
NAMESPACE   NAME              CLASS    HOSTS              AGE
production  orders-ingress    nginx    api.example.com    45d
production  orders-v2-ingress nginx    api.example.com    2m
production  users-ingress     nginx    api.example.com    90d
production  api-base-ingress  nginx    api.example.com    60d
```

多个 Ingress 资源指向同一个主机 `api.example.com`。`orders-v2-ingress` 创建于 2 分钟前——正好在部署窗口内。

### 第 2 步：查看问题 Ingress 的详情

```bash
kubectl describe ingress orders-v2-ingress -n production
```

```
Name:             orders-v2-ingress
Namespace:        production
Annotations:      nginx.ingress.kubernetes.io/rewrite-target: /api/v1/$2
                  kubernetes.io/ingress.class: nginx
Rules:
  Host              Path                                        Backends
  ----              ----                                        --------
  api.example.com   /api/v2/orders(/|$)(.*)                     orders-v2:8080 (10.244.1.20:8080)
```

注解一眼就看出了问题：`rewrite-target: /api/v1/$2`。这明显错了——它指向的是旧的 API v1 路径。

### 第 3 步：检查其他 Ingress 资源是否存在冲突

```bash
kubectl describe ingress api-base-ingress -n production
```

```
Name:             api-base-ingress
Namespace:        production
Annotations:      nginx.ingress.kubernetes.io/rewrite-target: /
Rules:
  Host              Path                                        Backends
  ----              ----                                        --------
  api.example.com   /api/v2(/.*)                                orders-v2:8080
  api.example.com   /api/v1(/.*)                                orders-v1:8080
```

`api-base-ingress` 有一个通配路径 `/api/v2(/.*)`，同样匹配原本打算走更具体的 `/api/v2/orders` 的请求。由于 NGINX 按特定顺序处理 location 块，且这个更宽泛的路径存在于一个**独立**的 Ingress 资源中，路由结果完全取决于控制器如何将这些规则合并到 nginx.conf 中。

### 第 4 步：查看 nginx-ingress 控制器日志

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=ingress-nginx --tail 100
```

```
I0609 14:00:15.123456       7 controller.go:181] "Configuration changes detected, triggering reload"
I0609 14:00:15.456789       7 controller.go:225] "Backend successfully reloaded"
I0609 14:00:15.456890       7 controller.go:234] "Reload completed successfully in 0.342s"
W0609 14:00:18.789012       7 controller.go:198] "Ingress re-queued due to path ordering conflict detected between 'production/orders-v2-ingress' and 'production/api-base-ingress'"
```

找到了——一条关于**路径排序冲突**的**警告**。但它只是警告，以 `WARNING` 级别记录，不是错误。控制器仍然成功完成了重载，因为 NGINX 配置在语法上是有效的。

### 第 5 步：查看实际 nginx.conf

```bash
kubectl exec -n kube-system nginx-ingress-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -A 20 "location /api/v2"
```

```nginx
# 来自 api-base-ingress —— 较宽泛的路径排在最前面
location ~* "^/api/v2(/.*)" {
    set $namespace      "production";
    set $backend        "orders-v2";
    set $service_port   "8080";
    rewrite /api/v2(/.*) / break;
    proxy_pass http://upstream_balancer;
}

# 来自 orders-v2-ingress —— 具体路径排在第二位（无效）
location ~* "^/api/v2/orders(/|$)(.*)" {
    set $namespace      "production";
    set $backend        "orders-v2";
    set $service_port   "8080";
    rewrite /api/v2/orders(/|$)(.*) /api/v1/$2 break;
    proxy_pass http://upstream_balancer;
}
```

两个关键问题显而易见：

1. **排序问题**：通配路径 `/api/v2(/.*)` 的 location 块出现在更具体的 `/api/v2/orders` 块**之前**。NGINX 处理正则 location（`~*`）时**按它们在配置中出现的顺序**——**第一个匹配的胜出**。因此 `/api/v2/orders/123` 首先匹配了 `/api/v2(/.*)`，永远到达不了 orders 专用的 location 块。

2. **重写 Bug**：即使匹配到了专用块，`rewrite-target: /api/v1/$2` 也会将路径重写为 `/api/v1/orders/123`——指向 **v1 服务路径**，而非 v2。这是从旧 Ingress 配置复制粘贴导致的错误。

### 第 6 步：从集群内外分别测试

```bash
# 从集群外部——404 错误
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/orders/123
# 404

# 从集群内部（直接访问服务）——正常工作
kubectl exec -n production deploy/orders-v2 -- curl -s http://localhost:8080/orders/123
# {"orderId": "123", "status": "processing", "v2Field": "new_schema"}
```

确认了服务本身是健康的。问题出在 Ingress 层。

```bash
# 测试命中宽泛路径时的行为
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/health
# 200（被 base ingress 正确路由）

# 测试具体路径
curl -v -H "Host: api.example.com" https://<ingress-ip>/api/v2/orders/123
# > GET /api/v2/orders/123 HTTP/2
# > Host: api.example.com
# >
# < HTTP/2 404
# < 
```

### 第 7 步：检查重写行为

```bash
# 进入 nginx-ingress pod 检查重写规则
kubectl exec -n kube-system nginx-ingress-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -B5 -A5 "rewrite.*/api/v1"
```

```nginx
rewrite /api/v2/orders(/|$)(.*) /api/v1/$2 break;
```

路径被重写为 `/api/v1/$2` 而非 `/v2/$2` 或 `/orders/$2`。当 NGINX 将重写后的请求代理到 orders-v2 时，服务收到的是 `/api/v1/orders/123`——这是一个它不提供的路径，导致应用返回 404。

---

## 根因

本次故障由两个叠加的错误导致：

### 错误 1：rewrite-target 注解的复制粘贴错误

新的 `orders-v2-ingress` Ingress 的 `rewrite-target` 注解是从旧的 orders-v1 Ingress 复制过来的：

```yaml
# orders-v2-ingress.yaml（错误——从 v1 配置复制粘贴而来）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-v2-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /api/v1/$2  # BUG: 应该是 /v2/$2
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v2/orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-v2
            port:
              number: 8080
```

正确的注解应为：

```yaml
    nginx.ingress.kubernetes.io/rewrite-target: /v2/$2
```

使用错误的注解时：
- 请求：`/api/v2/orders/123`
- 捕获的 `$2`：`orders/123`
- 重写为：`/api/v1/orders/123`（注意：v1！）
- 代理到 orders-v2：`/api/v1/orders/123`
- 结果：404——orders-v2 没有任何 `/api/v1/` 的路由

### 错误 2：两个 Ingress 资源之间的路径重叠

`api-base-ingress` 有一个宽泛的路径 `/api/v2(/.*)`，与 `orders-v2-ingress` 中更具体的 `/api/v2/orders(/|$)(.*)` 重叠。nginx-ingress 控制器根据自身的排序算法将宽泛的路径放在 nginx.conf 的前面，这意味着宽泛路径会**首先匹配所有**发往 `/api/v2/*` 的请求。

这导致了两个问题：
1. 发往 `/api/v2/orders/123` 的请求先被 base ingress 捕获
2. base ingress 的 `rewrite-target: /` 会剥离整个前缀——请求到达 orders-v2 时变成了 `/orders/123`（去掉了 `/api/v2`）。这对某些路径"碰巧"能工作，但对具有复杂子路由的路径则不行

然而，因为 base ingress 的重写目标是 `/`（剥离前缀），部分请求确实到达了 orders-v2 pod——这解释了**60% 的部分成功率**。混乱的根源在于哪个 location 块优先匹配，这取决于请求路径和 NGINX location 排序。

### 为什么 Ingress Controller 没有有效告警

控制器确实记录了警告：

```
W0609 14:00:18.789012 path ordering conflict detected between 'production/orders-v2-ingress' and 'production/api-base-ingress'
```

但是：
- 这是**警告**，不是错误——重载继续进行
- 在 `kubectl apply` 时没有准入 webhook 来拒绝重叠路径
- 在繁忙的控制器中，这条警告被淹没在数百行其他日志中
- 大多数团队不会在部署期间实时监控 ingress 控制器日志

```yaml
# api-base-ingress.yaml（已存在——截获流量的通配规则）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-base-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v2(/.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-v2
            port:
              number: 8080
```

---

## 解决方案

### 紧急修复（14:08——第一响应人）

1. **修正 rewrite-target 注解**：

```bash
kubectl edit ingress orders-v2-ingress -n production
```

```yaml
# 修复：将 rewrite-target 从 /api/v1/$2 改为 /v2/$2
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /v2/$2
```

2. **应用修正后的 Ingress**：

```bash
kubectl apply -f orders-v2-ingress.yaml
```

```
ingress.networking.k8s.io/orders-v2-ingress configured
```

3. **验证 NGINX 重载**：

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=ingress-nginx --tail 20
```

```
I0609 14:08:22.456789       7 controller.go:181] "Configuration changes detected, triggering reload"
I0609 14:08:22.789012       7 controller.go:225] "Backend successfully reloaded"
I0609 14:08:22.789123       7 controller.go:234] "Reload completed successfully in 0.287s"
```

4. **验证修复**：

```bash
# 验证具体端点现在正常工作
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/orders/123
# 200

# 验证宽泛路径仍然正常工作
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/health
# 200

# 验证旧 v1 端点不受影响
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v1/orders/123
# 200
```

5. **修复后检查 nginx.conf**：

```bash
kubectl exec -n kube-system nginx-ingress-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -A 15 "location ~\* \"\^/api/v2/orders"
```

```nginx
# 现在具体路径出现在配置中，且 rewrite 正确
location ~* "^/api/v2/orders(/|$)(.*)" {
    set $namespace      "production";
    set $backend        "orders-v2";
    set $service_port   "8080";
    rewrite /api/v2/orders(/|$)(.*) /v2/$2 break;
    proxy_pass http://upstream_balancer;
}
```

### 回滚计划（如果修复失败）

如果注解修复未能解决问题，回退方案是恢复到之前已知正常的 v1 配置：

```bash
# 回滚：重新部署之前正常工作的 v1 ingress
kubectl apply -f orders-v1-ingress.yaml

# 或者直接删除新 Ingress
kubectl delete ingress orders-v2-ingress -n production
```

### 长期预防措施

#### 1. 应用前验证 Ingress 配置

`ingress-nginx` 的 `kubectl` 插件提供了 `validate` 命令：

```bash
# 安装插件
kubectl krew install ingress-nginx

# 应用前验证 Ingress 资源
kubectl ingress-nginx validate --ingress orders-v2-ingress.yaml
```

但这只检查语法和基本结构——它不会检测与现有 Ingress 资源的路径重叠。这需要自定义验证步骤。

#### 2. 在 CI/CD 流水线中添加 Ingress 测试

先部署到 staging 环境，再执行基于 curl 的测试，确认无误后再推送到生产环境：

```yaml
# CI/CD 流水线步骤 —— ingress 冒烟测试
stages:
  - deploy-staging
  - ingress-smoke-test
  - deploy-production

ingress-smoke-test:
  script:
    - kubectl apply -f ingress/staging/
    - sleep 5  # 等待重载
    - |
      endpoints=(
        "/api/v2/orders/123"
        "/api/v2/orders"
        "/api/v2/health"
        "/api/v1/orders/456"
      )
      for ep in "${endpoints[@]}"; do
        status=$(curl -s -o /dev/null -w "%{http_code}" https://staging.example.com$ep)
        if [ "$status" != "200" ] && [ "$status" != "201" ]; then
          echo "FAIL: $ep returned $status"
          exit 1
        fi
        echo "PASS: $ep → $status"
      done
```

#### 3. 使用金丝雀发布实现流量逐步迁移

nginx-ingress controller 支持基于金丝雀权重的流量分流：

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-v2-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% 流量
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v2/orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-v2
            port:
              number: 8080
```

这会将爆炸半径限制在 10% 的用户，使问题暴露出来而不至于导致全面故障。

#### 4. 监控 Ingress 指标

将以下 nginx-ingress 指标添加到监控面板：

| 指标 | 说明 | 告警阈值 |
|---|---|---|
| `nginx_ingress_controller_requests` | 按状态码统计的请求数 | 4xx 率 > 5% 或 5xx > 1% |
| `nginx_ingress_controller_nginx_last_reload_successful` | 最近一次 NGINX 重载是否成功 | `0`（失败） |
| `nginx_ingress_controller_config_last_reload_successful` | 配置重载是否成功 | `0` |
| `nginx_ingress_controller_requests_seconds` | 请求延迟 | p99 > 2s |
| `nginx_ingress_controller_ssl_expire_time_seconds` | 证书过期时间 | < 30 天 |

本可立即捕获此问题的关键指标：`nginx_ingress_controller_requests` 在 14:02 显示 4xx 激增，但值班工程师最初将其误判为"客户端错误"，直到数据损坏告警触发。

#### 5. 部署流水线中的路径唯一性验证

添加预部署脚本检查路径重叠：

```bash
#!/bin/bash
# ingress-path-validator.sh
# 检查新 Ingress 与现有 Ingress 之间的路径重叠

NEW_INGRESS=$1
NEW_PATH=$(yq eval '.spec.rules[0].http.paths[0].path' "$NEW_INGRESS")
NEW_HOST=$(yq eval '.spec.rules[0].host' "$NEW_INGRESS")

echo "验证路径：$NEW_PATH，主机：$NEW_HOST"

# 获取现有 Ingress
kubectl get ingress -A -o yaml | yq eval '.items[] | select(.spec.rules[].host == "'$NEW_HOST'") | .spec.rules[].http.paths[].path' - |
while read existing_path; do
  if [[ "$NEW_PATH" == "$existing_path" ]] || [[ "$NEW_PATH" =~ ^${existing_path%\(.*} ]]; then
    echo "警告：检测到路径重叠！"
    echo "  新增：$NEW_PATH"
    echo "  现有：$existing_path"
    echo "  请在应用前审查！"
  fi
done
```

#### 6. 使用 OPA / Kubewarden 策略治理 Ingress

使用 Open Policy Agent（OPA）或 Kubewarden 来强制执行 Ingress 最佳实践：

```rego
# OPA 策略：拒绝同一主机上的重叠路径
package kubernetes.ingress

deny[msg] {
    input.request.kind.kind == "Ingress"
    host := input.request.object.spec.rules[_].host
    path := input.request.object.spec.rules[_].http.paths[_].path
    
    existing := data.kube_ingresses[host][_]
    existing != input.request.object.metadata.name
    existing_path := data.kube_ingresses[host][existing][_]
    
    regex.match(existing_path, path)
    msg := sprintf("路径 '%s' 与主机 '%s' 上的现有路径 '%s' 重叠", [path, host, existing_path])
}
```

---

## 经验教训

### 问题出在哪里

1. **注解中的复制粘贴错误对验证是不可见的**：注解 `rewrite-target: /api/v1/$2` 在语法上是完全有效的。没有 linter、没有验证器、没有准入控制器标记它。问题的第一个迹象是生产环境中的 404 错误。

2. **Ingress 资源之间的路径重叠不会被拒绝**：Kubernetes 允许多个 Ingress 资源在同一主机上定义路径。nginx-ingress 控制器会合并它们，但当路径重叠时，合并顺序是非确定性的。控制器会记录一条警告，但这很容易被忽视。

3. **缺少部署前的集成测试**：新的 Ingress 直接应用到生产环境，没有经过 staging 验证。在 staging 中做一个简单的 curl 测试本可以同时捕获路径重叠和错误的 rewrite-target。

4. **没有金丝雀或灰度发布**：Ingress 变更瞬间影响 100% 的流量。如果配置了金丝雀权重，爆炸半径本可以限制在一小部分用户。

5. **监控粒度不足**：团队监控了 5xx 错误，但没有对 4xx 激增或 ingress 重载警告设置告警。`ingress-nginx` 的重载警告对监控系统完全不可见。

### 我们改进了什么

- Ingress 变更现在需要在推送到生产环境之前**在 staging 部署并进行基于 curl 的冒烟测试**
- 所有 Ingress 注解需经过**同行评审清单**（rewrite-target、ssl-redirect、use-regex 等）
- **Ingress 路径重叠检测**已集成到 CI 流水线
- 监控现已包括 **4xx 速率告警**和 **ingress-nginx 重载失败告警**
- 团队入职培训包含**Ingress 调试实操工作坊**，覆盖 `nginx.conf` 检查流程
- 对所有影响现有路径的 Ingress 变更，标准化了**基于金丝雀的发布流程**

---

## 总结

本次故障展示了一个错误的注解，加上未检测到的路径重叠，如何悄然破坏生产流量：

| 问题 | 症状 | 修复 |
|---|---|---|
| `rewrite-target: /api/v1/$2`（复制粘贴错误） | 请求被重写到错误的 API 版本路径 | 修正为 `rewrite-target: /v2/$2` |
| 路径重叠：`/api/v2(/.*)` vs `/api/v2/orders(/|$)(.*)` | 通配路径优先匹配，绕过具体 Ingress 规则 | 重新排序路径、合并重叠规则，或使用路径唯一性验证 |
| 缺少语义正确性的准入验证 | 配置成功重载，尽管存在逻辑错误 | 添加 `kubectl ingress-nginx validate` 和自定义路径重叠检查 |
| Ingress 变更没有金丝雀发布 | 100% 流量立即受到影响 | 使用 `canary-weight` 注解进行灰度发布 |

### 攻击链

```
部署 orders-v2 及其 Ingress
    ↓
复制粘贴错误：rewrite-target → /api/v1/$2（应为 /v2/$2）
    ↓
路径重叠：api-base-ingress 中的 /api/v2(/.*) 优先匹配
    ↓
部分请求命中错误的 location 块 → 重写错误 → 404
其他请求命中通配规则 → 前缀被剥离 → 路由到 v1（数据污染）
    ↓
60% 的部分成功率掩盖了问题的严重性
    ↓
5 分钟后检测到数据损坏 → 事故确认
```

### 修复前后配置对比

| 维度 | 修复前（故障状态） | 修复后（正常状态） |
|---|---|---|
| `rewrite-target` | `/api/v1/$2`（版本错误） | `/v2/$2`（正确） |
| orders-v2-ingress 路径 | `/api/v2/orders(/|$)(.*)` | `/api/v2/orders(/|$)(.*)`（未变） |
| api-base-ingress 路径 | `/api/v2(/.*)`（重叠） | `/api/v2/health(/|$)(.*)`（收窄，无重叠） |
| CI 验证 | 无 | 路径重叠检查 + curl 冒烟测试 |
| 监控 | 仅 5xx | 4xx 速率 + 重载状态 + 路径冲突警告 |
| 发布策略 | 直接 apply | 金丝雀权重 10% → 50% → 100% |

**核心教训**：一个能通过 `nginx -t` 并成功重载的 Ingress 配置并不一定正确。语义错误——错误的重写目标、路径重叠、正则表达式的配置错误——会产生语法正确的 NGINX 配置，却将流量路由到错误的地方。唯一的防御是部署前验证、staging 集成测试、金丝雀发布以及对 HTTP 状态码进行细粒度监控（而非仅仅监控错误率）的组合。
