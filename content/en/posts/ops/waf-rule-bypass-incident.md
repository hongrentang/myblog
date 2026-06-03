---
title: "WAF Rule Bypass — When Your Web Application Firewall Missed the Obvious"
date: 2026-06-03
weight: 100310
slug: "waf-rule-bypass-incident"
tags: ["kubernetes", "security", "waf", "bypass", "troubleshooting"]
categories: ["Security"]
description: "A WAF bypass incident — how an attacker circumvented ModSecurity rules using HTTP parameter pollution and chunked transfer encoding tricks, leading to SQL injection on a production database"
keywords: "waf bypass incident, modsecurity crs bypass, http parameter pollution, waf rule evasion, kubernetes waf bypass"
draft: false
featured: true
cover:
  image: ""
  caption: "WAF Rule Bypass — Incident Investigation"
---

# WAF Rule Bypass — When Your Web Application Firewall Missed the Obvious

## Common Search Queries

- WAF bypass techniques modsecurity
- OWASP CRS SQL injection bypass
- HTTP parameter pollution WAF evasion
- chunked transfer encoding WAF bypass
- modsecurity rule evasion real world example

---

## The Incident

**Environment**: Kubernetes v1.28, Nginx Ingress Controller with ModSecurity enabled and OWASP CRS v3.3.5, MariaDB 10.11 on a dedicated node, single-region production cluster.

**Time**: Tuesday 03:14. PagerDuty alerted on a sudden spike in database CPU usage (from 15% to 97% in under 2 minutes) and an unusually high number of error-level SQL queries in the application logs.

**Initial Symptoms**:

- API response times jumped from ~120ms to over 12s
- Database connection pool exhausted within 90 seconds
- Application pods started returning 503 errors
- The `/api/products/search` endpoint showed 4,700 requests in 3 minutes from a single IP block

```bash
# Initial observability check
kubectl top pods -n production | grep api
# Output showed CPU throttling across all 4 API replicas

# Database connection pool status
mysqladmin -u monitoring -p status -h db-primary.production.svc | grep Threads
# Threads: 487  (pool max was 150)
```

**Impact**:

- Application unavailable for approximately 27 minutes
- Customer-facing storefront returned 503 errors
- Estimated 12,000 abandoned shopping sessions
- No evidence of data exfiltration, but 1,847 records were queried from the `users` table via injection
- Incident cost: approximately $47,000 in lost revenue plus 8 engineering hours for response

---

## Background

The application was a typical e-commerce platform running on Kubernetes. A Nginx Ingress Controller sat at the edge, terminating TLS and forwarding traffic to the API service. The ingress had ModSecurity with OWASP Core Rule Set enabled in "anomaly scoring" mode — or so everyone believed.

The relevant ingress configuration:

```yaml
# ingress.yaml (simplified)
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

The backend API was built with Express.js (Node.js 20) and used a raw SQL query builder for the search functionality. The search endpoint looked like this:

```javascript
// routes/search.js - the vulnerable endpoint
router.get('/products/search', async (req, res) => {
  const { q, category, sort, page, limit } = req.query;

  // Direct interpolation into SQL — no parameterized query
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

The WAF was deployed to catch exactly this kind of vulnerability. But on that Tuesday morning, it didn't.

---

## Investigation

### Step 1: Confirm the Attack Vector

We grabbed the ingress access logs and filtered by the affected endpoint during the incident window:

```bash
kubectl logs -n ingress-nginx -l app=ingress-nginx --tail=50000 | \
  grep "/api/products/search" | \
  awk '{print $1, $4, $7, $9, $11}' | \
  head -20

# Sample output:
# 203.0.113.42 [03/Jun/2026:03:14:12] /api/products/search 200 0.427
# 203.0.113.42 [03/Jun/2026:03:14:13] /api/products/search 200 0.381
# (repeated from same IP)
```

The requests were all returning 200 — the WAF was not blocking anything. We needed to see what the actual payloads looked like.

```bash
# Extract the full query string from the logs
kubectl logs -n ingress-nginx -l app=ingress-nginx --tail=200000 | \
  grep "/api/products/search" | \
  head -5 | \
  jq -R 'split(" ") | {uri: .[6], status: .[8], ua: .[11]}'
```

### Step 2: Reconstruct the Attack Payload

We decoded the URL-encoded query parameters from the logs:

```bash
# Reconstructed URL from raw log entry
# Original encoded request:
# GET /api/products/search?q=phone&category=Electronics&sort=price&page=1&limit=20

# But the malicious requests had additional parameters injected:
# GET /api/products/search?q=phone&category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--&sort=price&page=1&limit=20
```

The attacker was injecting SQL via the `category` parameter. But why didn't the WAF catch it?

### Step 3: Check ModSecurity Audit Logs

```bash
# Check ModSecurity audit logs inside the ingress controller pod
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /var/log/modsec_audit.log | tail -50

# No entries found for the attack timeframe!
# The WAF literally had nothing to report — it never saw the malicious payload
```

This was the key finding: **the ModSecurity audit log was empty for those requests**. The WAF was not processing the malicious payloads at all.

### Step 4: Compare Legitimate Request vs Attack Request

We captured a side-by-side comparison of the request structures:

```bash
# Legitimate request - WAF processes normally
curl -v "https://api.example.com/api/products/search?q=phone&category=Electronics&sort=price&page=1&limit=20"
# Returns: 200 OK, valid products
# ModSecurity audit: entry logged, score=0

# Attack request - WAF does NOT process the injection
curl -v "https://api.example.com/api/products/search?q=phone&category=Electronics&sort=price&page=1&limit=20&category=Electronics' UNION SELECT * FROM users--"
# Returns: 200 OK, users table data returned
# ModSecurity audit: NO ENTRY — WAF completely blind to this request
```

The difference? The attack request had **two `category` parameters**. The WAF only inspected the first one. The backend application read the second one.

### Step 5: Verify the WAF Bypass Mechanism

We tested with different parameter duplication patterns:

```bash
# Test 1: Single parameter — blocked correctly
curl -v "https://api.example.com/api/products/search?category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--"
# Response: 403 Forbidden — blocked by WAF (ModSecurity anomaly score > threshold)

# Test 2: Duplicated parameter — bypass successful
curl -v "https://api.example.com/api/products/search?category=Electronics&category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--"
# Response: 200 OK — SQL injection succeeded, WAF did not inspect second parameter

# Test 3: Different parameter ordering
curl -v "https://api.example.com/api/products/search?category=Electronics'%20UNION%20SELECT%20*%20FROM%20users--&category=Electronics"
# Response: 403 Forbidden — blocked (first parameter was malicious, WAF saw it)
```

We had confirmed **HTTP Parameter Pollution (HPP)** as the bypass technique. But this was only the beginning — further analysis revealed the attacker also used a second bypass method.

### Step 6: Discover the Chunked Transfer Encoding Bypass

Looking deeper at the access logs, some requests were POST to an endpoint that normally only accepted GET. These POST requests used chunked transfer encoding:

```bash
# From ingress logs — a POST with chunked encoding
# 203.0.113.42 [03/Jun/2026:03:16:44] "POST /api/products/search HTTP/1.1" 200 0.512
# Transfer-Encoding: chunked
# Content-Type: application/x-www-form-urlencoded
```

We decoded the chunked request body:

```bash
# The attacker sent the payload in two chunks
# Chunk 1: "q=phone&category=Electronics"
# Chunk 2: "&category=Electronics' UNION SELECT * FROM users--"

# The WAF reassembled and inspected Chunk 1 independently
# But passed Chunk 2 through without re-inspection
```

The attacker combined **two bypass techniques**:
1. **HTTP Parameter Pollution** — duplicate parameter names where only the second was malicious
2. **Chunked Transfer Encoding Splitting** — split the payload across chunks so no single chunk triggered a rule match

### Step 7: Check WAF Configuration

```bash
# Check ModSecurity configuration in the ingress controller
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/modsecurity/modsecurity.conf | grep -E "(SecRuleEngine|SecRequestBodyAccess|SecResponseBodyAccess|REQUEST-900-EXCLUSION-RULES-BEFORE-CRS)"

# Key findings:
# SecRuleEngine On
# SecRequestBodyAccess On
# SecResponseBodyAccess Off
```

The configuration looked correct at a high level. But we found the critical flaw:

```bash
# Check the OWASP CRS configuration file
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/owasp-crs/crs-setup.conf | grep -i "REQUEST-903.9001-DRUPAL-EXCLUSION-RULES"

# Unexpected finding: Some exclusion rules were loaded from a legacy config
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/owasp-crs/rules/REQUEST-903.9001-DRUPAL-EXCLUSION-RULES
```

The Drupal exclusion ruleset was active — a leftover from a previous CMS migration. This ruleset contained exceptions that disabled certain SQL injection checks for specific parameter patterns, including `category`.

### Step 8: Verify the Complete Bypass Chain

```bash
# Full reconstruction of the bypass
# Step 1: WAF inspects request
# Step 2: Sees only first occurrence of each parameter (Nginx default behavior)
# Step 3: Drupal exclusion rules suppress SQLi checks on 'category' parameter
# Step 4: Second 'category' parameter passes through un-inspected
# Step 5: Express.js reads the LAST occurrence of duplicate parameters (different from Nginx!)
# Step 6: SQL injection executed on the database
```

We had the full picture: the attacker exploited a **mismatch** between how:
- **Nginx/ModSecurity** handles duplicate parameters (reads **first** occurrence)
- **Express.js** handles duplicate parameters (reads **last** occurrence)

---

## Root Cause

1. **HTTP Parameter Pollution vulnerability**: Nginx/ModSecurity processed only the first occurrence of duplicate query parameters, while Express.js consumed the last occurrence. This discrepancy allowed the attacker to hide malicious payloads in the second occurrence

2. **Chunked Transfer Encoding bypass**: ModSecurity was configured to inspect each chunk independently rather than reconstructing and inspecting the full request body. The attacker split the payload across chunks to evade signature detection

3. **Orphaned WAF exclusion rules**: The OWASP CRS Drupal exclusion ruleset (`REQUEST-903.9001-DRUPAL-EXCLUSION-RULES`) was accidentally left active after a CMS migration. It contained broad SQL injection exclusions for specific parameters, including `category`

4. **No request normalization validation**: The team never tested whether the WAF's parameter parsing matched the backend's parameter parsing. The assumption "the WAF sees what the app sees" was incorrect

5. **Insufficient ModSecurity audit logging**: The ModSecurity audit log was writing to a local volume inside the pod. When the pod restarted (as it had 3 days prior), the log file was lost. If the logs had been persisted to a centralized location, the bypass would have been detected earlier

6. **No layer 7 WAF rule testing in CI/CD**: WAF rules were deployed as part of the ingress chart but were never tested against known attack patterns in staging. A simple smoke test would have caught the exclusion rule issue

---

## Resolution

### Emergency (Immediate)

```bash
# 1. Block the attacker IP at the network edge
kubectl patch ingress api-ingress -n production -p '{
  "metadata": {
    "annotations": {
      "nginx.ingress.kubernetes.io/block-cidrs": "203.0.113.42/32"
    }
  }
}'

# 2. Kill all active database connections from the API user
mysql -h db-primary.production.svc -u root -p \
  -e "SELECT GROUP_CONCAT(CONCAT('KILL ', id, ';') SEPARATOR ' ') 
      FROM information_schema.processlist 
      WHERE user = 'api_user' AND id != CONNECTION_ID();" \
  | tail -1 | mysql -h db-primary.production.svc -u root -p

# 3. Force restart all API pods to clear any in-memory state
kubectl rollout restart deploy/api-service -n production

# 4. Enable query logging on the database to assess damage
mysql -h db-primary.production.svc -u root -p \
  -e "SET GLOBAL general_log = ON; SET GLOBAL log_output = 'TABLE';"
```

### Remove the Orphaned Drupal Exclusion Ruleset

```yaml
# modsecurity-crs-config.yaml — remove Drupal exclusion rules
apiVersion: v1
kind: ConfigMap
metadata:
  name: owasp-crs-config
  namespace: ingress-nginx
data:
  crs-setup.conf: |
    # OWASP ModSecurity Core Rule Set setup file
    # https://coreruleset.org/

    # Default action and engine mode
    SecDefaultAction "phase:1,log,auditlog,pass"
    SecDefaultAction "phase:2,log,auditlog,pass"

    # Anomaly scoring
    SecAction \
      "id:900000,\
      phase:1,\
      nolog,\
      pass,\
      t:none,\
      setvar:tx.anomaly_score_threshold=5"

    # REMOVED: Include REQUEST-903.9001-DRUPAL-EXCLUSION-RULES
    # (Leftover from CMS migration — was disabling SQLi checks)

    # Parameter pollution protection
    SecRule ARGS_GET_NAMES|ARGS_POST_NAMES "@pm category q sort page" \
      "id:100001,\
      phase:2,\
      deny,\
      status:403,\
      msg:'Duplicate parameter detected — possible HPP attack',\
      ver:'custom/1.0',\
      logdata:'Duplicated param: %{MATCHED_VAR_NAME}',\
      chain"
      SecRule MATCHED_VAR "@rx ^(.*)$" \
        "chain"
        SecRule &ARGS_GET_NAMES|&ARGS_POST_NAMES "@gt 1"

    # ... rest of configuration continues
```

### Fix Parameter Parsing — Add HPP Detection in Nginx

```nginx
# In the ingress nginx template — add parameter duplication detection
# /etc/nginx/template/nginx.tmpl (relevant section)

location /api/ {
    # HTTP Parameter Pollution detection
    # Block requests where any query parameter appears more than once
    if ($args ~* "(\w+)=.*&\1=") {
        return 403;
    }

    proxy_pass http://api-service.production.svc:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
}
```

### Fix the Backend — Parameterized Queries

```javascript
// routes/search.js — fixed with parameterized queries
router.get('/products/search', async (req, res) => {
  const { q, category, sort, page, limit } = req.query;

  // Validate and sanitize
  const safePage = Math.max(1, parseInt(page) || 1);
  const safeLimit = Math.min(100, Math.max(1, parseInt(limit) || 20));
  const safeOffset = (safePage - 1) * safeLimit;

  // Allowed sort columns whitelist
  const allowedSorts = ['p.created_at', 'p.price', 'p.name', 'p.rating'];
  const safeSort = allowedSorts.includes(sort) ? sort : 'p.created_at';

  // Parameterized query — no string interpolation
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

### Fix Chunked Transfer Encoding Inspection

```bash
# In the ModSecurity config — enforce full body inspection
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  sed -i 's/SecRequestBodyInMemoryLimit .*/SecRequestBodyInMemoryLimit 2097152/' \
    /etc/nginx/modsecurity/modsecurity.conf

kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  sed -i 's/SecRequestBodyLimit .*/SecRequestBodyLimit 2097152/' \
    /etc/nginx/modsecurity/modsecurity.conf

# Add rule to reject chunked encoding for API endpoints
# (unless absolutely needed for streaming)
```

```yaml
# Additional ModSecurity rule to block chunked encoding on non-streaming endpoints
rules:
  - id: 100002
    phase: 1
    deny:
      status: 406
    msg: "Chunked Transfer-Encoding blocked for non-streaming API endpoint"
    transformation: none
    tag: "custom/waf"
    var:
      - REQUEST_HEADERS:Transfer-Encoding
    pattern: "chunked"
```

### Add Centralized ModSecurity Audit Logging

```yaml
# modsecurity-logging.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: modsecurity-logging
  namespace: ingress-nginx
data:
  modsecurity.conf: |
    # Audit log configuration
    SecAuditEngine RelevantOnly
    SecAuditLog /dev/stdout
    SecAuditLogFormat JSON
    SecAuditLogParts ABIJDEFHZ

    # Send to stdout for collection by Fluentd/Vector
  vector.toml: |
    # Vector configuration to ship ModSecurity logs to Elasticsearch
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

### Add WAF Rule Validation to CI/CD

```yaml
# .github/workflows/waf-test.yml
name: WAF Rule Validation
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
      - name: Start ModSecurity test harness
        run: |
          docker compose -f tests/waf/docker-compose.yml up -d
      - name: Run OWASP CRS bypass tests
        run: |
          # Test HTTP parameter pollution scenarios
          curl -s -o /dev/null -w "%{http_code}" \
            "http://localhost:8080/api/search?q=test&q=test'%20OR%201=1--" \
            | grep -q 403 || (echo "FAIL: HPP bypass not blocked" && exit 1)

          # Test SQL injection with known bypass vectors
          curl -s -o /dev/null -w "%{http_code}" \
            "http://localhost:8080/api/search?category=Electronics'+UNION+SELECT+*+FROM+users--" \
            | grep -q 403 || (echo "FAIL: SQLi not blocked" && exit 1)

          # Test chunked encoding bypass
          curl -s -o /dev/null -w "%{http_code}" \
            -H "Transfer-Encoding: chunked" \
            -d "q=phone&category=E" \
            http://localhost:8080/api/search \
            | grep -q 403 || (echo "FAIL: Chunked bypass not blocked" && exit 1)

      - name: Notify on failure
        if: failure()
        run: |
          echo "WAF rule validation failed. Review the configuration changes."
```

---

## Lessons Learned

- **The WAF and the backend must agree on parameter parsing**: If Nginx reads the first duplicate parameter and Express.js reads the last, you have a security gap. Test parameter handling at every layer of the stack
- **Chunked encoding is a blind spot for many WAFs**: Ensure your WAF reassembles request bodies before inspection. A rule that inspects one chunk at a time is not a rule at all
- **Orphaned exclusion rules are silent killers**: During migrations, audit your WAF configurations. An exclusion rule from a CMS you no longer run can silently disable protections for years
- **ModSecurity audit logs should be centralized**: If the pod restarts and the logs were on a local volume, your forensic evidence disappears. Ship them to a central store immediately
- **WAF rules need automated testing**: A regression test suite for WAF rules is as important as unit tests for application code. Without it, you deploy configurations you haven't validated
- **Defense in depth matters**: If the API had used parameterized queries instead of string interpolation, the WAF bypass would have been irrelevant. A WAF is a safety net, not a primary defense

---

## Summary

The attack chain:

```
Attacker probes /api/products/search endpoint
→ Identifies SQL injection vulnerability in 'category' parameter
→ Tests WAF: single malicious parameter → 403 blocked
→ Tests parameter duplication: category=Electronics&category=' UNION ...
→ Nginx/ModSecurity inspects only FIRST occurrence → passes (benign)
→ Express.js reads LAST occurrence → SQL injection executed
→ Also uses chunked encoding to split payload across chunks
→ Chunked inspection fails to reassemble → payload not detected
→ Orphaned Drupal exclusion rules further suppress WAF rules on 'category'
→ Complete bypass achieved — WAF sees safe data, app receives exploit
→ Database CPU spikes to 97%, connection pool exhausted
→ Application goes down for 27 minutes
```

Three layers of defense failed simultaneously: the WAF didn't parse correctly, the exclusion rules were misconfigured, and the application didn't use parameterized queries. The fix required changes in all three layers — WAF configuration, CI/CD testing, and application code.

```
Root Cause Tree:

           WAF Bypass Incident
                   │
         ┌─────────┼─────────┐
         │         │         │
     HPP in    Chunked    Orphaned
     Nginx     Encoding   Exclusion
     (reads    (chunk     Rules
     first)    split)    (Drupal)
         │         │         │
         └─────────┼─────────┘
                   │
          ┌────────┴────────┐
          │                 │
    Backend reads     No parameterized
    last param        queries in API
          │                 │
          └────────┬────────┘
                   │
          SQL Injection
          Executed on DB
```
