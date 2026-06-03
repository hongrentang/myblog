---
title: "WAF 规则绕过事件——当 Web 应用防火墙对明显攻击视而不见"
date: 2026-06-03
weight: 100310
slug: "waf-rule-bypass-incident"
tags: ["kubernetes", "security", "waf", "bypass", "troubleshooting"]
categories: ["Security"]
description: "一次 WAF 绕过事件复盘——攻击者利用 HTTP 参数污染和分块传输编码技巧绕过了 ModSecurity 规则，最终对生产数据库成功执行了 SQL 注入"
keywords: "WAF 绕过事件, ModSecurity CRS 绕过, HTTP 参数污染, WAF 规则绕过, Kubernetes WAF 绕过"
draft: false
featured: true
cover:
  image: ""
  caption: "WAF 规则绕过——安全事件排查"
---

# WAF 规则绕过事件——当 Web 应用防火墙对明显攻击视而不见

## 常见搜索词

- WAF 绕过技术 ModSecurity
- OWASP CRS SQL 注入绕过
- HTTP 参数污染 WAF 绕过
- 分块传输编码 WAF 绕过
- ModSecurity 规则绕过真实案例

---

## 故障经过

**环境**: Kubernetes v1.28, Nginx Ingress Controller 启用 ModSecurity + OWASP CRS v3.3.5, MariaDB 10.11 运行在专属节点, 单区域生产集群。

**时间**: 周二 03:14。PagerDuty 告警：数据库 CPU 使用率突然飙升（从 15% 到 97% 不到 2 分钟），应用日志中出现大量错误级别的 SQL 查询。

**初始症状**:

- API 响应时间从 ~120ms 飙升到 12s 以上
- 数据库连接池在 90 秒内耗尽
- 应用 Pod 开始返回 503 错误
- `/api/products/search` 端点在 3 分钟内收到来自同一 IP 段的 4,700 个请求

```bash
# 初始可观测性检查
kubectl top pods -n production | grep api
# 输出显示所有 4 个 API 副本均出现 CPU 节流

# 数据库连接池状态
mysqladmin -u monitoring -p status -h db-primary.production.svc | grep Threads
# Threads: 487  (连接池最大为 150)
```

**影响**:

- 应用大约 27 分钟不可用
- 面向客户的商城页面返回 503 错误
- 估计约 12,000 个购物会话被中断
- 未发现数据泄露证据，但攻击者通过 SQL 注入查询了 `users` 表的 1,847 条记录
- 事故损失：约 $47,000 的收入损失 + 8 个工程小时的响应时间

---

## 背景

该应用是一个典型的运行在 Kubernetes 上的电商平台。Nginx Ingress Controller 位于边缘，终止 TLS 并将流量转发到 API 服务。Ingress 启用了 ModSecurity 和 OWASP Core Rule Set 的"异常评分"模式——至少团队是这么认为的。

相关的 Ingress 配置：

```yaml
# ingress.yaml（简化版）
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/enable-owasp-core-rules: "true"
    nginx.ingress.kubernetes.io/modsecurity-transaction-id: "$request_id"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

后端 API 使用 Express.js（Node.js 20）构建，搜索功能使用了原始的 SQL 查询拼接。搜索端点的代码如下：

```javascript
// routes/search.js — 存在漏洞的端点
router.get('/products/search', async (req, res) => {
  const { q, category, sort, page, limit } = req.query;

  // 直接将参数拼接到 SQL 中——没有使用参数化查询
  const sql = `
    SELECT p.*, u.email as seller_email
    FROM products p
    JOIN users u ON p.seller_id = u.id
    WHERE p.name LIKE '%${q}%'
    ${category ? `AND p.category = '${category}'` : ''}
    ORDER BY ${sort || 'p.created_at'} DESC
    LIMIT ${parseInt(limit) || 20} OFFSET ${(parseInt(page) - 1) * (parseInt(limit) || 20)}
  `;

  const results = await pool.query(sql);
  res.json({ data: results.rows, total: results.rowCount });
});
```

部署 WAF 正是为了捕捉这类漏洞。但在那个周二早晨，它没有做到。

---

## 排查过程

### 第一步：确认攻击向量

我们提取了 Ingress 访问日志，并按受影响端点和时间窗口进行过滤：

```bash
kubectl logs -n ingress-nginx -l app=ingress-nginx --tail=50000 | \
  grep "/api/products/search" | \
  awk '{print $1, $4, $7, $9, $11}' | \
  head -20

# 输出示例：
# 203.0.113.42 [03/Jun/2026:03:14:12] /api/products/search 200 0.427
# 203.0.113.42 [03/Jun/2026:03:14:13] /api/products/search 200 0.381
# （来自同一 IP 的重复请求）
```

所有请求都返回 200——WAF 没有阻止任何请求。我们需要查看实际的请求载荷。

```bash
# 从日志中提取完整的查询字符串
kubectl logs -n ingress-nginx -l app=ingress-nginx --tail=200000 | \
  grep "/api/products/search" | \
  head -5 | \
  jq -R 'split(" ") | {uri: .[6], status: .[8], ua: .[11]}'
```

### 第二步：重构攻击载荷

我们从日志中解码了 URL 编码的查询参数：

```bash
# 从原始日志解码出的 URL
# 正常请求：
# GET /api/products/search?q=phone&category=Electronics&sort=price&page=1&limit=20

# 恶意请求中多了额外的参数注入：
# GET /api/products/search?q=phone&category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--&sort=price&page=1&limit=20
```

攻击者通过 `category` 参数注入 SQL。但为什么 WAF 没有捕捉到？

### 第三步：检查 ModSecurity 审计日志

```bash
# 检查 Ingress Controller Pod 中的 ModSecurity 审计日志
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /var/log/modsec_audit.log | tail -50

# 攻击时间窗口内没有找到任何条目！
# WAF 根本没有记录——它从未看到恶意载荷
```

这是关键发现：**针对这些请求，ModSecurity 审计日志是空的**。WAF 根本没有处理恶意载荷。

### 第四步：对比正常请求与攻击请求

我们捕获了请求结构的对比：

```bash
# 正常请求 - WAF 正常处理
curl -v "https://api.example.com/api/products/search?q=phone&category=Electronics&sort=price&page=1&limit=20"
# 返回: 200 OK, 有效产品列表
# ModSecurity 审计: 有条目, 评分为 0

# 攻击请求 - WAF 未处理注入
curl -v "https://api.example.com/api/products/search?q=phone&category=Electronics&sort=price&page=1&limit=20&category=Electronics' UNION SELECT * FROM users--"
# 返回: 200 OK, 返回了 users 表数据
# ModSecurity 审计: 无条目 — WAF 对这个请求完全不可见
```

区别在哪？攻击请求中有 **两个 `category` 参数**。WAF 只检查了第一个。后端应用读取的是第二个。

### 第五步：验证 WAF 绕过机制

我们测试了不同的参数重复模式：

```bash
# 测试 1: 单一参数 — 被正确拦截
curl -v "https://api.example.com/api/products/search?category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--"
# 响应: 403 Forbidden — 被 WAF 拦截（ModSecurity 异常评分超过阈值）

# 测试 2: 重复参数 — 绕过成功
curl -v "https://api.example.com/api/products/search?category=Electronics&category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--"
# 响应: 200 OK — SQL 注入成功, WAF 未检查第二个参数

# 测试 3: 不同参数顺序
curl -v "https://api.example.com/api/products/search?category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--&category=Electronics"
# 响应: 403 Forbidden — 被拦截（第一个参数是恶意的, WAF 看到了）
```

我们确认了 **HTTP 参数污染（HPP）** 作为绕过技术。但这只是开始——进一步分析显示攻击者还使用了第二种绕过方法。

### 第六步：发现分块传输编码绕过

深入分析访问日志发现，有些请求是 POST 到通常只接受 GET 的端点。这些 POST 请求使用了分块传输编码：

```bash
# 来自 Ingress 日志 — 使用分块编码的 POST
# 203.0.113.42 [03/Jun/2026:03:16:44] "POST /api/products/search HTTP/1.1" 200 0.512
# Transfer-Encoding: chunked
# Content-Type: application/x-www-form-urlencoded
```

我们解码了分块请求体：

```bash
# 攻击者将载荷分成两个块发送
# 块 1: "q=phone&category=Electronics"
# 块 2: "&category=Electronics' UNION SELECT * FROM users--"

# WAF 独立重建并检查了块 1（正常）
# 但直接将块 2 传递了过去而没有重新检查
```

攻击者结合了 **两种绕过技术**：
1. **HTTP 参数污染** — 重复参数名称，只有第二个是恶意的
2. **分块传输编码拆分** — 将载荷分散到多个块中，使得单个块不会触发任何规则匹配

### 第七步：检查 WAF 配置

```bash
# 检查 Ingress Controller 中的 ModSecurity 配置
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/modsecurity/modsecurity.conf | grep -E "(SecRuleEngine|SecRequestBodyAccess|SecResponseBodyAccess|REQUEST-900-EXCLUSION-RULES-BEFORE-CRS)"

# 关键发现：
# SecRuleEngine On
# SecRequestBodyAccess On
# SecResponseBodyAccess Off
```

从高层次看，配置似乎正确。但我们发现了关键缺陷：

```bash
# 检查 OWASP CRS 配置文件
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/owasp-crs/crs-setup.conf | grep -i "REQUEST-903.9001-DRUPAL-EXCLUSION-RULES"

# 意外发现：遗留配置中加载了一些排除规则
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/owasp-crs/rules/REQUEST-903.9001-DRUPAL-EXCLUSION-RULES
```

Drupal 排除规则集竟然被激活了——这是之前 CMS 迁移留下的残留。该规则集包含一些例外规则，对特定参数模式禁用了某些 SQL 注入检查，其中就包括 `category` 参数。

### 第八步：验证完整的绕过链

```bash
# 完整的绕过还原
# 步骤 1: WAF 检查请求
# 步骤 2: 只看到每个参数的第一次出现（Nginx 默认行为）
# 步骤 3: Drupal 排除规则抑制了对 'category' 参数的 SQL 注入检查
# 步骤 4: 第二个 'category' 参数未经检查直接放行
# 步骤 5: Express.js 读取重复参数的最后一个值（与 Nginx 不同！）
# 步骤 6: SQL 注入在数据库上成功执行
```

我们获得了完整的图景：攻击者利用了 **三个层面的不一致性**：
- **Nginx/ModSecurity** 处理重复参数的方式：读取 **第一个** 出现
- **Express.js** 处理重复参数的方式：读取 **最后一个** 出现
- **OWASP CRS 排除规则** 意外禁用了关键检查

---

## 根因

1. **HTTP 参数污染漏洞**：Nginx/ModSecurity 只处理重复查询参数的第一次出现，而 Express.js 消费最后一次出现。这一差异允许攻击者在第二次出现中隐藏恶意载荷

2. **分块传输编码绕过**：ModSecurity 被配置为独立检查每个块，而不是重建并检查完整的请求体。攻击者将载荷拆分为多个块以绕过签名检测

3. **遗留的 WAF 排除规则**：OWASP CRS 的 Drupal 排除规则集（`REQUEST-903.9001-DRUPAL-EXCLUSION-RULES`）在 CMS 迁移后意外保持激活。其中包含针对特定参数（包括 `category`）的广泛 SQL 注入排除规则

4. **未进行请求规范化验证**：团队从未测试过 WAF 的参数解析是否与后端的参数解析一致。"WAF 看到的就是应用看到的"这个假设是不正确的

5. **ModSecurity 审计日志不足**：ModSecurity 审计日志写入 Pod 内的本地卷。当 Pod 重启（3 天前发生过），日志文件丢失。如果日志被持久化到集中存储，绕过行为可能更早被发现

6. **CI/CD 中缺少 WAF 规则测试**：WAF 规则作为 Ingress Chart 的一部分部署，但从未在预发布环境中针对已知攻击模式进行测试。一次简单的冒烟测试就能发现排除规则的问题

---

## 解决方案

### 紧急处置

```bash
# 1. 在网络边缘封禁攻击者 IP
kubectl patch ingress api-ingress -n production -p '{
  "metadata": {
    "annotations": {
      "nginx.ingress.kubernetes.io/block-cidrs": "203.0.113.42/32"
    }
  }
}'

# 2. 杀掉 API 用户的所有活跃数据库连接
mysql -h db-primary.production.svc -u root -p \
  -e "SELECT GROUP_CONCAT(CONCAT('KILL ', id, ';') SEPARATOR ' ') 
      FROM information_schema.processlist 
      WHERE user = 'api_user' AND id != CONNECTION_ID();" \
  | tail -1 | mysql -h db-primary.production.svc -u root -p

# 3. 强制重启所有 API Pod 以清除内存中的残留状态
kubectl rollout restart deploy/api-service -n production

# 4. 在数据库上启用查询日志以评估损失
mysql -h db-primary.production.svc -u root -p \
  -e "SET GLOBAL general_log = ON; SET GLOBAL log_output = 'TABLE';"
```

### 移除遗留的 Drupal 排除规则集

```yaml
# modsecurity-crs-config.yaml — 移除 Drupal 排除规则
apiVersion: v1
kind: ConfigMap
metadata:
  name: owasp-crs-config
  namespace: ingress-nginx
data:
  crs-setup.conf: |
    # OWASP ModSecurity Core Rule Set 配置文件
    # https://coreruleset.org/

    # 默认动作和引擎模式
    SecDefaultAction "phase:1,log,auditlog,pass"
    SecDefaultAction "phase:2,log,auditlog,pass"

    # 异常评分
    SecAction \
      "id:900000,\
      phase:1,\
      nolog,\
      pass,\
      t:none,\
      setvar:tx.anomaly_score_threshold=5"

    # 已移除：Include REQUEST-903.9001-DRUPAL-EXCLUSION-RULES
    #（CMS 迁移遗留物——禁用了某些 SQLi 检查）

    # 参数污染保护
    SecRule ARGS_GET_NAMES|ARGS_POST_NAMES "@pm category q sort page" \
      "id:100001,\
      phase:2,\
      deny,\
      status:403,\
      msg:'检测到重复参数——可能的 HPP 攻击',\
      ver:'custom/1.0',\
      logdata:'重复参数: %{MATCHED_VAR_NAME}',\
      chain"
      SecRule MATCHED_VAR "@rx ^(.*)$" \
        "chain"
        SecRule &ARGS_GET_NAMES|&ARGS_POST_NAMES "@gt 1"

    # ...后续配置继续
```

### 修复参数解析——在 Nginx 中添加 HPP 检测

```nginx
# 在 Ingress nginx 模板中——添加参数重复检测
# /etc/nginx/template/nginx.tmpl（相关段落）

location /api/ {
    # HTTP 参数污染检测
    # 阻止任何查询参数出现多次的请求
    if ($args ~* "(\w+)=.*&\1=") {
        return 403;
    }

    proxy_pass http://api-service.production.svc:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

### 修复后端——使用参数化查询

```javascript
// routes/search.js — 使用参数化查询修复
router.get('/products/search', async (req, res) => {
  const { q, category, sort, page, limit } = req.query;

  // 验证和清理
  const safePage = Math.max(1, parseInt(page) || 1);
  const safeLimit = Math.min(100, Math.max(1, parseInt(limit) || 20));
  const safeOffset = (safePage - 1) * safeLimit;

  // 允许的排序列白名单
  const allowedSorts = ['p.created_at', 'p.price', 'p.name', 'p.rating'];
  const safeSort = allowedSorts.includes(sort) ? sort : 'p.created_at';

  // 参数化查询——不使用字符串拼接
  const sql = `
    SELECT p.*, u.email as seller_email
    FROM products p
    JOIN users u ON p.seller_id = u.id
    WHERE p.name LIKE $1
    ${category ? 'AND p.category = $2' : ''}
    ORDER BY ${safeSort} DESC
    LIMIT $3 OFFSET $4
  `;

  const params = [`%${q}%`];
  if (category) params.push(category);
  params.push(safeLimit, safeOffset);

  const results = await pool.query(sql, params);
  res.json({ data: results.rows, total: results.rowCount });
});
```

### 修复分块传输编码检查

```bash
# 在 ModSecurity 配置中——强制全请求体检查
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  sed -i 's/SecRequestBodyInMemoryLimit .*/SecRequestBodyInMemoryLimit 2097152/' \
    /etc/nginx/modsecurity/modsecurity.conf

kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  sed -i 's/SecRequestBodyLimit .*/SecRequestBodyLimit 2097152/' \
    /etc/nginx/modsecurity/modsecurity.conf

# 添加规则以拒绝非流式 API 端点的分块编码
```

```yaml
# 额外的 ModSecurity 规则，阻止非流式端点的分块编码
rules:
  - id: 100002
    phase: 1
    deny:
      status: 406
    msg: "非流式 API 端点阻止分块传输编码"
    transformation: none
    tag: "custom/waf"
    var:
      - REQUEST_HEADERS:Transfer-Encoding
    pattern: "chunked"
```

### 添加集中式 ModSecurity 审计日志

```yaml
# modsecurity-logging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: modsecurity-logging
  namespace: ingress-nginx
data:
  modsecurity.conf: |
    # 审计日志配置
    SecAuditEngine RelevantOnly
    SecAuditLog /dev/stdout
    SecAuditLogFormat JSON
    SecAuditLogParts ABIJDEFHZ

    # 输出到 stdout，由 Fluentd/Vector 采集
  vector.toml: |
    # Vector 配置——将 ModSecurity 日志发送到 Elasticsearch
    [sources.modsec]
    type = "file"
    paths = ["/proc/1/fd/1"]

    [transforms.modsec_parse]
    type = "remap"
    inputs = ["modsec"]
    source = '''
    . = parse_json!(.message)
    '''

    [sinks.elasticsearch]
    type = "elasticsearch"
    inputs = ["modsec_parse"]
    endpoints = ["http://elasticsearch.logging.svc:9200"]
    index = "modsecurity-%Y-%m-%d"
```

### 在 CI/CD 中添加 WAF 规则验证

```yaml
# .github/workflows/waf-test.yml
name: WAF 规则验证
on:
  pull_request:
    paths:
      - "infra/ingress/**"
      - "charts/ingress/**"

jobs:
  test-waf-rules:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: 启动 ModSecurity 测试框架
        run: |
          docker compose -f tests/waf/docker-compose.yml up -d
      - name: 运行 OWASP CRS 绕过测试
        run: |
          # 测试 HTTP 参数污染场景
          curl -s -o /dev/null -w "%{http_code}" \
            "http://localhost:8080/api/search?q=test&q=test'%20OR%201=1--" \
            | grep -q 403 || (echo "失败：HPP 绕过未被拦截" && exit 1)

          # 测试已知绕过向量的 SQL 注入
          curl -s -o /dev/null -w "%{http_code}" \
            "http://localhost:8080/api/search?category=Electronics'+UNION+SELECT+*+FROM+users--" \
            | grep -q 403 || (echo "失败：SQLi 未被拦截" && exit 1)

          # 测试分块编码绕过
          curl -s -o /dev/null -w "%{http_code}" \
            -H "Transfer-Encoding: chunked" \
            -d "q=phone&category=E" \
            http://localhost:8080/api/search \
            | grep -q 403 || (echo "失败：分块编码绕过未被拦截" && exit 1)

      - name: 失败时通知
        if: failure()
        run: |
          echo "WAF 规则验证失败。请检查配置变更。"
```

---

## 经验教训

- **WAF 和后端必须对参数解析达成一致**：如果 Nginx 读取第一个重复参数，而 Express.js 读取最后一个，就存在安全缺口。测试每一层的参数处理行为
- **分块编码是许多 WAF 的盲点**：确保你的 WAF 在检查前重建请求体。一次只检查一个块的规则根本不是规则
- **遗留的排除规则是无声的杀手**：在迁移过程中审计你的 WAF 配置。来自你不再运行的 CMS 的排除规则可以静默禁用保护好多年
- **ModSecurity 审计日志应该集中化**：如果 Pod 重启而日志在本地卷上，你的取证证据就消失了。立即将它们发送到中央存储
- **WAF 规则需要自动化测试**：WAF 规则的回归测试套件与应用代码的单元测试同样重要。没有它，你就是在部署未经验证的配置
- **纵深防御至关重要**：如果 API 使用了参数化查询而不是字符串拼接，WAF 绕过就无关紧要了。WAF 是安全网，不是主要防御手段

---

## 总结

攻击链路：

```
攻击者探测 /api/products/search 端点
→ 识别 'category' 参数中的 SQL 注入漏洞
→ 测试 WAF：单一恶意参数 → 403 被拦截
→ 测试参数重复：category=Electronics&category=' UNION ...
→ Nginx/ModSecurity 只检查第一个出现 → 通过（无害）
→ Express.js 读取最后一个出现 → SQL 注入执行
→ 同时使用分块编码将载荷拆分到多个块中
→ 块检查未能重建 → 载荷未被检测
→ 遗留的 Drupal 排除规则进一步抑制了对 'category' 的 WAF 规则
→ 完全绕过实现 — WAF 看到安全数据，应用收到恶意载荷
→ 数据库 CPU 飙升到 97%，连接池耗尽
→ 应用宕机 27 分钟
```

三个防御层同时失效：WAF 没有正确解析参数，排除规则配置错误，应用没有使用参数化查询。修复需要在所有三个层面进行更改——WAF 配置、CI/CD 测试和应用代码。

```
根因分析树：

           WAF 规则绕过事件
                   │
         ┌─────────┼─────────┐
         │         │         │
   Nginx 中   分块传输    Drupal 遗留
   HPP 漏洞   编码拆分   排除规则
  (读第一个)  (块拆分)   (活跃状态)
         │         │         │
         └─────────┼─────────┘
                   │
          ┌────────┴────────┐
          │                 │
    后端读取最后     API 中未使用
    一个参数        参数化查询
          │                 │
          └────────┬────────┘
                   │
          SQL 注入在数据库上执行
```
