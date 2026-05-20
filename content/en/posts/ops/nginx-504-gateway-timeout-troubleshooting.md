---
title: "Nginx 504 Gateway Timeout — From Blaming Upstream to Fixing Keepalive"
date: 2026-05-20
weight: 100090
slug: "nginx-504-gateway-timeout-troubleshooting"
tags: ["nginx", "network", "troubleshooting"]
categories: ["网络"]
description: "Complete guide to troubleshooting Nginx 504 Gateway Timeout — from misdiagnosing upstream latency to finding connection pool and keepalive misconfig"
keywords: "nginx 504, gateway timeout, upstream timed out, nginx keepalive, proxy_read_timeout"
draft: false
featured: true
cover:
  image: "/images/nginx-504-banner.svg"
  caption: "Nginx 504 Gateway Timeout Troubleshooting"
---

# Nginx 504 Gateway Timeout — From Blaming Upstream to Fixing Keepalive

## Symptoms

API endpoints started returning 504 intermittently. Not all requests — some worked fine, others hung for 30 seconds then returned 504.

```bash
curl -I https://api.example.com/v1/orders
# Waits 30 seconds then:
# HTTP/2 504
```

From Nginx access logs:

```bash
tail -f /var/log/nginx/access.log | grep " 504 "
```

```
10.0.0.1 - - [20/May/2026:14:30:15 +0800] "GET /v1/orders" 504 0 "-" "curl/7.68" 0.050 30.121
```

`upstream_response_time` 30s, `request_time` 30s. Nginx waited 30 seconds for the upstream, then timed out.

**Impact**: Core API intermittently unavailable, frontend errors, users can't place orders.

## Investigation

### Wrong Turn 1: Blamed the Upstream

504 means "upstream didn't respond in time." First thought: check the backend.

```bash
# Bypass Nginx, hit backend directly
curl -w "time_total: %{time_total}s\n" http://127.0.0.1:8080/v1/orders
```

```
<normal response>
time_total: 0.085s
```

85ms. Then why does Nginx see 30s timeouts?

```bash
for i in $(seq 1 10); do
    curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" http://127.0.0.1:8080/v1/orders
done
```

```
200 0.082s
200 0.091s
... (all under 100ms)
```

Backend is fine.

### Wrong Turn 2: Increased proxy_read_timeout

Maybe the timeout is too short:

```bash
grep -r "proxy_read_timeout" /etc/nginx/
```

```
proxy_read_timeout 30s;
```

Matches the 504 behavior. Bump to 60s:

```bash
sed -i 's/proxy_read_timeout 30s;/proxy_read_timeout 60s;/' /etc/nginx/conf.d/default.conf
nginx -s reload
```

Test again — still 504 after 60 seconds. It just made users wait longer without reducing the 504 **rate**.

**Lesson**: If the upstream responds in 100ms, a 30s timeout is more than enough. Increasing it won't help — the issue is that some requests never connect, not that they're slow.

### The Real Discovery: Connection Reuse

Bypassing Nginx works every time. Through Nginx, intermittent 504s. The problem is in the **connection** between Nginx and upstream.

Check the upstream config:

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

`proxy_http_version 1.1` with empty `Connection` header enables keepalive. But:

```bash
grep -r "keepalive" /etc/nginx/
```

No `keepalive` directive exists.

This is the problem: HTTP/1.1 keepalive is enabled between Nginx and upstream, but no `keepalive` connection pool is configured. Under high concurrency, connections get closed instead of reused — flooding the system with TIME_WAIT.

```bash
ss -s
```

```
TCP:   9876 (estab 234, closed 6543, timewait 3210, ...)
```

3210 TIME_WAIT connections.

```bash
ss -tan | grep 8080 | awk '{print $1}' | sort | uniq -c
```

```
   8 ESTAB
  35 TIME-WAIT
```

35 TIME_WAIT connections to the backend. New connections block waiting for port release.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct cause | Insufficient connection pool between Nginx and upstream — TIME_WAIT accumulation blocks new connections |
| Config flaw | HTTP/1.1 keepalive enabled without `keepalive` directive — connections close instead of being reused |
| Trigger | Traffic spike created connection surge |

Bottom line: **HTTP/1.1 keepalive without `keepalive` directive is functionally similar to no keepalive at all.**

## Solutions

### Option A: Quick Fix — Increase worker_connections (Stop the Bleeding)

```nginx
# /etc/nginx/nginx.conf
worker_connections 4096;
```

```bash
nginx -s reload
```

Temporary relief, not a fix.

### Option B: Configure upstream keepalive (Recommended)

```nginx
upstream backend {
    server 127.0.0.1:8080;
    keepalive 128;  # Keep 128 idle keepalive connections
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # Sensible timeouts (not blindly increased)
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 10s;
    }
}
```

```bash
nginx -t && nginx -s reload
```

Verify TIME_WAIT drop:

```bash
ss -tan | grep 8080 | awk '{print $1}' | sort | uniq -c
```

```
  42 ESTAB
   3 TIME-WAIT
```

TIME_WAIT dropped from 35 to 3.

### Option C: Kernel Tuning (Production)

```bash
cat >> /etc/sysctl.conf << 'EOF'
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_fin_timeout = 15
EOF
sysctl -p
```

**Option comparison**:

| Option | Scenario | Effect | Permanent? |
|--------|----------|--------|------------|
| A. Increase worker_connections | Emergency | Temporary relief | ❌ |
| B. Configure keepalive | General use | Significantly reduces connection overhead | ✅ |
| C. Kernel tuning | High-traffic production | Reduces TIME_WAIT impact | ✅辅助 |

### Verify Recovery

```bash
# 1. Confirm no more 504s
tail -f /var/log/nginx/access.log | grep -v " 504 "

# 2. Test endpoint
curl -I https://api.example.com/v1/orders
# Expected: 200

# 3. Monitor TIME_WAIT
watch -n 2 'ss -tan | grep 8080 | awk "{print \$1}" | sort | uniq -c'
```

✅ **Recovery criteria**:
- No 504 responses
- TIME_WAIT connections drop to single digits
- Frontend恢复正常

## Long-Term Prevention

```bash
# 1. Monitor upstream response time
# Prometheus: nginx_upstream_response_time_seconds > 10
# Alert before 504 happens

# 2. Weekly TIME_WAIT check
# Threshold: TIME_WAIT > 1000 triggers investigation

# 3. Load test new services before production
wrk -t4 -c200 -d30s https://api.example.com/v1/orders
# Verify zero 50x errors
```

## What I Learned

1. **504 doesn't mean upstream is slow.** Bypass Nginx and test the backend directly before touching any timeout config. Blindly increasing `proxy_read_timeout` only makes users wait longer without fixing the error rate.

2. **HTTP/1.1 keepalive without `keepalive` is a trap.** Enabling HTTP/1.1 connection reuse without configuring the `keepalive` directive means connections still get closed and recreated — no better than not using keepalive at all.

3. **TIME_WAIT is the canary in the coal mine.** `ss -s` and `ss -tan | uniq -c` should be the first commands when debugging connection issues. Don't let it accumulate.

4. **Bypass the proxy to isolate blame.** Curling the backend directly is a 5-second test that immediately tells you whether the problem is in the app or the proxy layer. This single check would have saved 30 minutes of wrong turns.
