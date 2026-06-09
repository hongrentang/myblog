---
title: "Ingress Routing Failure — When Traffic Went to the Wrong Service and Nobody Noticed"
date: 2026-06-09
weight: 100450
slug: "ingress-routing-failure"
tags: ["kubernetes", "networking", "troubleshooting", "ingress", "nginx"]
categories: ["Troubleshooting"]
description: "A Kubernetes Ingress routing incident — how a misconfigured Ingress path rewrite rule caused production traffic to be routed to the wrong backend service, leading to data corruption and a 2-hour outage"
keywords: "kubernetes ingress troubleshooting, nginx ingress path rewrite, ingress routing wrong service, k8s ingress 404, ingress configuration debugging"
draft: false
featured: true
cover:
  image: ""
  caption: "Ingress Routing Failure — Troubleshooting"
---

# Ingress Routing Failure — When Traffic Went to the Wrong Service and Nobody Noticed

## Common Search Queries

If you're landing here from a search, this post covers:

- kubernetes ingress routing to wrong service troubleshooting
- nginx ingress rewrite-target annotation wrong path
- kubernetes ingress 404 after new deployment
- ingress path regex conflict multiple rules overlapping
- nginx ingress controller traffic partially goes to old backend
- kubectl ingress not routing correctly after update
- ingress-nginx validate configuration before apply

---

## The Incident

**Environment**: K8S v1.28, nginx-ingress-controller v1.8, 3 microservices — orders-v1, orders-v2, users. Ingress-NGINX installed via Helm in `kube-system` namespace.

**Time**: Tuesday 14:00, during the scheduled deployment window.

**Symptoms**: The monitoring board showed a sudden spike in 404 errors on the `/api/v2/orders` endpoint. At first glance, the new orders-v2 service appeared healthy — pods were running, liveness probes passed, and metrics showed successful internal processing. But the truth was more insidious: some requests to `/api/v2/orders` were being silently routed to the **old** orders-v1 backend instead of the new orders-v2. Data inconsistencies began appearing in the database as orders were split across two versions of the service, and partial success rates hovered around 60%.

```bash
# Alerts triggered
Service: orders-v2  |  Error: GET /api/v2/orders/123 → 404 Not Found  (response from ingress)
Service: orders-v2  |  Error: POST /api/v2/orders → 201 Created but body processed by orders-v1 (wrong schema)
Database: order-svc  |  Warning: 38% of orders missing expected fields from v2 schema
```

**Impact**: Approximately 40% of order traffic was degraded. The checkout flow partially failed, producing corrupted order records with missing fields. Some orders were created by orders-v1 but later expected by orders-v2, causing cascading failures in downstream services. A revenue-affecting incident was declared at 14:05, and the full recovery — including database repair — took approximately 2 hours.

---

## Background

### How nginx-ingress-controller Works

The NGINX Ingress Controller translates Kubernetes Ingress resources into an NGINX configuration file (`nginx.conf`) and reloads NGINX whenever the configuration changes. The data flow is:

```
Ingress Resource (YAML) → Ingress Controller (watch loop) → nginx.conf generation → NGINX reload → Traffic routing
```

When you create or update an Ingress resource, the controller:
1. Watches for changes via the Kubernetes API
2. Regenerates the NGINX configuration template
3. Validates the generated config with `nginx -t`
4. Reloads NGINX with the new configuration (if validation passes)

Crucially, step 3 only checks for **syntax errors** — not semantic correctness. A valid but wrong configuration will reload successfully.

### Path Matching Types

nginx-ingress supports three path types:

| Type | Behavior | Example |
|---|---|---|
| `Exact` | Matches the path exactly (`/api/v2/orders` matches only that path) | No prefix matching |
| `Prefix` | Matches paths starting with the given prefix (`/api` matches `/api/`, `/api/v1/`, etc.) | `/api` prefix |
| `ImplementationSpecific` | Defers to the NGINX-specific behavior — the path is treated as an NGINX location pattern (can contain regex) | `/api/v2/orders(/\|$)(.*)` |

The `ImplementationSpecific` type with regex patterns is where complexity — and bugs — originate.

### The rewrite-target Annotation

`nginx.ingress.kubernetes.io/rewrite-target` controls how the request path is rewritten before being sent to the backend service. Capture groups from the path pattern can be referenced as `$1`, `$2`, etc.:

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /$2
```

If your Ingress path is `/api/v2/orders(/|$)(.*)` and rewrite-target is `/$2`, then:
- Request: `/api/v2/orders/123` → captures `$2 = "orders/123"` → rewritten to `/orders/123`
- Request: `/api/v2/orders` → captures `$2 = ""` → rewritten to `/`

This is extremely powerful, but also extremely error-prone when copy-pasting between configurations.

### Ingress Controller Reload Mechanics

The nginx-ingress controller batches configuration changes within a `sync-period` (default 30 seconds). When an Ingress resource changes:

1. Controller detects the change via informer watch
2. Waits for the sync period to batch any additional changes
3. Regenerates the NGINX template
4. Runs `nginx -t` to validate syntax
5. Sends a reload signal (SIGHUP) to NGINX

The entire process is asynchronous. There is no built-in validation for path conflicts, routing overlap, or annotation correctness.

---

## Investigation

### Step 1: Check Ingress Resources

The first step — list all Ingress resources to understand the routing landscape:

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

Multiple Ingress resources were targeting the same host `api.example.com`. The `orders-v2-ingress` had been created 2 minutes ago — during the deployment window.

### Step 2: Describe the Problematic Ingress

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

The annotation jumped out: `rewrite-target: /api/v1/$2`. This looked wrong — it was rewriting to the old API v1 path.

### Step 3: Check Other Ingress Resources for Conflicts

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

The `api-base-ingress` had a catch-all `/api/v2(/.*)` path that also matched requests intended for the more specific `/api/v2/orders`. Since NGINX processes location blocks in a specific order, and this broader path existed in a **separate** Ingress resource, the routing outcome depended entirely on how the controller merged these rules into nginx.conf.

### Step 4: Check nginx-ingress Controller Logs

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=ingress-nginx --tail 100
```

```
I0609 14:00:15.123456       7 controller.go:181] "Configuration changes detected, triggering reload"
I0609 14:00:15.456789       7 controller.go:225] "Backend successfully reloaded"
I0609 14:00:15.456890       7 controller.go:234] "Reload completed successfully in 0.342s"
W0609 14:00:18.789012       7 controller.go:198] "Ingress re-queued due to path ordering conflict detected between 'production/orders-v2-ingress' and 'production/api-base-ingress'"
```

There it was — a **warning** about a path ordering conflict. But it was only a warning, logged at the `WARNING` level, not an error. The controller still completed the reload successfully because the NGINX configuration was syntactically valid.

### Step 5: Inspect the Actual nginx.conf

```bash
kubectl exec -n kube-system nginx-ingress-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -A 20 "location /api/v2"
```

```nginx
# From api-base-ingress — broader path appeared FIRST
location ~* "^/api/v2(/.*)" {
    set $namespace      "production";
    set $backend        "orders-v2";
    set $service_port   "8080";
    rewrite /api/v2(/.*) / break;
    proxy_pass http://upstream_balancer;
}

# From orders-v2-ingress — specific path appeared SECOND (no effect)
location ~* "^/api/v2/orders(/|$)(.*)" {
    set $namespace      "production";
    set $backend        "orders-v2";
    set $service_port   "8080";
    rewrite /api/v2/orders(/|$)(.*) /api/v1/$2 break;
    proxy_pass http://upstream_balancer;
}
```

Two critical problems visible:

1. **Ordering**: The catch-all `/api/v2(/.*)` location block appeared **before** the more specific `/api/v2/orders` block. NGINX processes regex locations (`~*`) in the order they appear in the config — the **first match wins**. So `/api/v2/orders/123` matched `/api/v2(/.*)` first and never reached the orders-specific block.

2. **Rewrite bug**: Even if the specific block had matched, `rewrite-target: /api/v1/$2` would rewrite the path to `/api/v1/orders/123` — pointing to the **v1 service path**, not v2. This was a copy-paste error from an old Ingress configuration.

### Step 6: Test From Inside and Outside the Cluster

```bash
# From outside the cluster — 404 errors
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/orders/123
# 404

# From inside the cluster (direct to service) — works fine
kubectl exec -n production deploy/orders-v2 -- curl -s http://localhost:8080/orders/123
# {"orderId": "123", "status": "processing", "v2Field": "new_schema"}
```

This confirmed the service itself was healthy. The problem was in the Ingress layer.

```bash
# Test what happens when hitting the broad path
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/health
# 200 (routed correctly by the base ingress)

# Test the specific path
curl -v -H "Host: api.example.com" https://<ingress-ip>/api/v2/orders/123
# > GET /api/v2/orders/123 HTTP/2
# > Host: api.example.com
# >
# < HTTP/2 404
# < 
```

### Step 7: Check the Rewrite Behavior

```bash
# Execute into the nginx-ingress pod and check the config for rewrite rules
kubectl exec -n kube-system nginx-ingress-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -B5 -A5 "rewrite.*/api/v1"
```

```nginx
rewrite /api/v2/orders(/|$)(.*) /api/v1/$2 break;
```

The path was being rewritten to `/api/v1/$2` instead of `/v2/$2` or `/orders/$2`. When NGINX proxied the rewritten request to orders-v2, the service received `/api/v1/orders/123` — a path it did not serve, resulting in a 404 from the application.

---

## Root Cause

The incident had two compounding failures:

### Failure 1: Copy-Paste Error in rewrite-target Annotation

The new Ingress `orders-v2-ingress` had a `rewrite-target` annotation copied from an older orders-v1 Ingress:

```yaml
# orders-v2-ingress.yaml (WRONG — copy-pasted from v1 config)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-v2-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /api/v1/$2  # BUG: should be /v2/$2
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

The correct annotation should have been:

```yaml
    nginx.ingress.kubernetes.io/rewrite-target: /v2/$2
```

With the buggy annotation:
- Request: `/api/v2/orders/123`
- Captured `$2`: `orders/123`
- Rewritten to: `/api/v1/orders/123` (note: v1!)
- Proxied to orders-v2: `/api/v1/orders/123`
- Result: 404 — orders-v2 has no `/api/v1/` routes

### Failure 2: Path Overlap Between Two Ingress Resources

The `api-base-ingress` had a broader path `/api/v2(/.*)` that overlapped with the more specific `/api/v2/orders(/|$)(.*)` in `orders-v2-ingress`. The nginx-ingress controller placed the broader path first in nginx.conf due to its own sorting algorithm, which meant the broader path matched first for ALL requests hitting `/api/v2/*`.

This caused two problems:
1. Requests to `/api/v2/orders/123` were caught by the base ingress first
2. The base ingress had `rewrite-target: /`, which stripped the entire prefix — the request reached orders-v2 as `/orders/123` (drop the `/api/v2`). This actually worked for some paths but not for others with complex sub-routing.

However, because the base ingress rewrite-target was `/` (stripping the prefix), some requests DID reach the orders-v2 pod — explaining the **partial success rate of 60%**. The chaos came from which location block matched first, which varied depending on request path and NGINX location ordering.

### Why the Ingress Controller Didn't Warn Effectively

The controller DID log a warning:

```
W0609 14:00:18.789012 path ordering conflict detected between 'production/orders-v2-ingress' and 'production/api-base-ingress'
```

But:
- It was a **warning**, not an error — the reload proceeded
- There was no admission webhook to reject overlapping paths at `kubectl apply` time
- The warning was buried among hundreds of other log lines in a busy controller
- Most teams do not monitor ingress controller logs in real-time during deployments

```yaml
# api-base-ingress.yaml (pre-existing — the catch-all that intercepted traffic)
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

## Resolution

### Emergency Fix (14:08 — First Responder)

1. **Correct the rewrite-target annotation**:

```bash
kubectl edit ingress orders-v2-ingress -n production
```

```yaml
# Fix: change rewrite-target from /api/v1/$2 to /v2/$2
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /v2/$2
```

2. **Apply the corrected Ingress**:

```bash
kubectl apply -f orders-v2-ingress.yaml
```

```
ingress.networking.k8s.io/orders-v2-ingress configured
```

3. **Verify NGINX reload**:

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=ingress-nginx --tail 20
```

```
I0609 14:08:22.456789       7 controller.go:181] "Configuration changes detected, triggering reload"
I0609 14:08:22.789012       7 controller.go:225] "Backend successfully reloaded"
I0609 14:08:22.789123       7 controller.go:234] "Reload completed successfully in 0.287s"
```

4. **Test the fix**:

```bash
# Verify the specific endpoint now works
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/orders/123
# 200

# Verify the broad path still works
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v2/health
# 200

# Verify old v1 endpoints are unaffected
curl -s -o /dev/null -w "%{http_code}" -H "Host: api.example.com" https://<ingress-ip>/api/v1/orders/123
# 200
```

5. **Check nginx.conf after fix**:

```bash
kubectl exec -n kube-system nginx-ingress-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -A 15 "location ~\* \"\^/api/v2/orders"
```

```nginx
# Now the specific path appeared with correct rewrite
location ~* "^/api/v2/orders(/|$)(.*)" {
    set $namespace      "production";
    set $backend        "orders-v2";
    set $service_port   "8080";
    rewrite /api/v2/orders(/|$)(.*) /v2/$2 break;
    proxy_pass http://upstream_balancer;
}
```

### Rollback Plan (If Fix Failed)

If the annotation fix didn't resolve the issue, the fallback was to roll back to the known-good v1 configuration:

```bash
# Rollback: redeploy the v1 ingress that was working before
kubectl apply -f orders-v1-ingress.yaml

# Or delete the new ingress entirely
kubectl delete ingress orders-v2-ingress -n production
```

### Long-Term Preventative Measures

#### 1. Validate Ingress Configurations Before Applying

The `ingress-nginx` plugin for `kubectl` provides a `validate` command:

```bash
# Install the plugin
kubectl krew install ingress-nginx

# Validate an Ingress resource before applying
kubectl ingress-nginx validate --ingress orders-v2-ingress.yaml
```

However, this only checks syntax and basic structure — it does not detect path overlaps with existing Ingress resources. For that, you need a custom validation step.

#### 2. Add Ingress Testing to CI/CD Pipeline

Deploy to a staging environment and run curl-based tests before promoting to production:

```yaml
# CI/CD pipeline step — ingress smoke test
stages:
  - deploy-staging
  - ingress-smoke-test
  - deploy-production

ingress-smoke-test:
  script:
    - kubectl apply -f ingress/staging/
    - sleep 5  # wait for reload
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

#### 3. Use Canary Releases for Gradual Traffic Migration

The nginx-ingress controller supports canary-based traffic splitting:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-v2-canary
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% of traffic
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

This would have limited the blast radius to 10% of users, making the problem visible without causing a full outage.

#### 4. Monitor Ingress Metrics

Add these nginx-ingress metrics to your monitoring stack:

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| `nginx_ingress_controller_requests` | Request count by status code | 4xx rate > 5% or 5xx > 1% |
| `nginx_ingress_controller_nginx_last_reload_successful` | Whether the last NGINX reload succeeded | `0` (failed) |
| `nginx_ingress_controller_config_last_reload_successful` | Whether config reload was successful | `0` |
| `nginx_ingress_controller_requests_seconds` | Request latency | p99 > 2s |
| `nginx_ingress_controller_ssl_expire_time_seconds` | Certificate expiry | < 30 days |

The key metric that would have caught this immediately: `nginx_ingress_controller_requests` showed a 4xx spike at 14:02, but the on-call engineer initially dismissed it as "client errors" before the data corruption alerts fired.

#### 5. Path Uniqueness Validation in Deployment Pipeline

Add a pre-deployment script that checks for overlapping paths:

```bash
#!/bin/bash
# ingress-path-validator.sh
# Check for overlapping paths between new ingress and existing ones

NEW_INGRESS=$1
NEW_PATH=$(yq eval '.spec.rules[0].http.paths[0].path' "$NEW_INGRESS")
NEW_HOST=$(yq eval '.spec.rules[0].host' "$NEW_INGRESS")

echo "Validating path: $NEW_PATH for host: $NEW_HOST"

# Get existing ingresses
kubectl get ingress -A -o yaml | yq eval '.items[] | select(.spec.rules[].host == "'$NEW_HOST'") | .spec.rules[].http.paths[].path' - |
while read existing_path; do
  if [[ "$NEW_PATH" == "$existing_path" ]] || [[ "$NEW_PATH" =~ ^${existing_path%\(.*} ]]; then
    echo "WARNING: Path overlap detected!"
    echo "  New:      $NEW_PATH"
    echo "  Existing: $existing_path"
    echo "  Review before applying!"
  fi
done
```

#### 6. OPA / Kubewarden Policies for Ingress Governance

Use Open Policy Agent (OPA) or Kubewarden to enforce Ingress best practices:

```rego
# OPA policy: deny overlapping paths on the same host
package kubernetes.ingress

deny[msg] {
    input.request.kind.kind == "Ingress"
    host := input.request.object.spec.rules[_].host
    path := input.request.object.spec.rules[_].http.paths[_].path
    
    existing := data.kube_ingresses[host][_]
    existing != input.request.object.metadata.name
    existing_path := data.kube_ingresses[host][existing][_]
    
    regex.match(existing_path, path)
    msg := sprintf("Path '%s' overlaps with existing path '%s' on host '%s'", [path, existing_path, host])
}
```

---

## Lessons Learned

### What Went Wrong

1. **Copy-paste errors in annotations are invisible to validation**: The `rewrite-target: /api/v1/$2` annotation was syntactically valid. No linter, no validator, no admission controller flagged it. The first indication of a problem was 404 errors in production.

2. **Path overlap between Ingress resources is not rejected**: Kubernetes allows multiple Ingress resources to define paths on the same host. The nginx-ingress controller merges them, but the merge order is non-deterministic when paths overlap. The controller logs a warning, but this is easy to miss.

3. **No pre-deploy integration testing**: The new Ingress was applied directly to production without staging verification. A simple curl test in staging would have caught both the path overlap and the incorrect rewrite-target.

4. **No canary or gradual rollout**: The Ingress change went to 100% of traffic instantly. If a canary weight had been configured, the blast radius would have been limited to a small percentage of users.

5. **Monitoring granularity was insufficient**: The team monitored 5xx errors but had no alert on 4xx rate spikes or ingress reload warnings. The `ingress-nginx` reload warning was invisible to the monitoring stack.

### What We Improved

- Ingress changes now require **staging deployment with curl-based smoke tests** before production rollout
- All Ingress annotations go through a **peer review checklist** (rewrite-target, ssl-redirect, use-regex, etc.)
- **Ingress path overlap detection** is integrated into the CI pipeline
- Monitoring now includes **4xx rate alerts** and **reload failure alerts** for ingress-nginx
- Team onboarding includes a **hands-on Ingress debugging workshop** covering the `nginx.conf` inspection workflow
- Standardized **canary-based rollout** for all Ingress changes affecting existing paths

---

## Summary

This incident demonstrates how a single incorrect annotation, combined with undetected path overlap, can silently corrupt production traffic:

| Issue | Symptom | Fix |
|---|---|---|
| `rewrite-target: /api/v1/$2` (copy-paste error) | Requests rewritten to wrong API version path | Correct to `rewrite-target: /v2/$2` |
| Path overlap: `/api/v2(/.*)` vs `/api/v2/orders(/|$)(.*)` | Catch-all matched first, routing bypassed specific Ingress rules | Reorder paths, consolidate overlapping rules, or use path uniqueness validation |
| No admission validation for semantic correctness | Config reloaded successfully despite logical errors | Add `kubectl ingress-nginx validate` and custom path overlap checks |
| No canary for Ingress changes | 100% traffic impacted immediately | Use `canary-weight` annotation for gradual rollout |

### Attack Chain

```
Deploy orders-v2 with Ingress
    ↓
Copy-paste error: rewrite-target → /api/v1/$2 (should be /v2/$2)
    ↓
Path overlap: /api/v2(/.*) in api-base-ingress matches first
    ↓
Some requests hit the wrong location block → rewritten incorrectly → 404
Other requests hit the catch-all → stripped prefix → routed to v1 (data corruption)
    ↓
60% partial success rate masked the severity
    ↓
Data corruption detected after 5 minutes → incident declared
```

### Before vs After Configuration Comparison

| Aspect | Before (Broken) | After (Fixed) |
|---|---|---|
| `rewrite-target` | `/api/v1/$2` (wrong version) | `/v2/$2` (correct) |
| Path in orders-v2-ingress | `/api/v2/orders(/|$)(.*)` | `/api/v2/orders(/|$)(.*)` (unchanged) |
| Path in api-base-ingress | `/api/v2(/.*)` (overlapping) | `/api/v2/heath(/|$)(.*)` (narrowed, no overlap) |
| CI validation | None | Path overlap check + curl smoke tests |
| Monitoring | 5xx only | 4xx rate + reload status + path conflict warnings |
| Rollout strategy | Direct apply | Canary weight 10% → 50% → 100% |

**Key takeaway**: An Ingress configuration that passes `nginx -t` and produces a successful reload is not necessarily correct. Semantic errors — wrong rewrite targets, path overlaps, and regex misconfigurations — produce valid NGINX configs that route traffic to the wrong places. The only defense is a combination of pre-deploy validation, staging integration tests, canary rollouts, and granular monitoring of HTTP status codes rather than just error rates.
