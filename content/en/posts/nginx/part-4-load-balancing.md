---
title: "Nginx from Beginner to Pro · Part 4: Load Balancing"
date: 2025-01-08
weight: 1104
draft: false
tags: ["nginx"]
---

## Why Load Balancing?

When a single server can't handle the traffic, you need multiple servers to share the load. Load balancing **distributes requests reasonably across multiple backend servers**.

```
                    ┌─ Server A (10.0.0.1:8080)
User → Nginx ────  ├─ Server B (10.0.0.2:8080)
                    ├─ Server C (10.0.0.3:8080)
                    └─ Server D (backup)
```

---

## Upstream Configuration

The core of load balancing is the `upstream` block:

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

---

## Load Balancing Algorithms

### 1. Round Robin (Default)

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

Requests are distributed one-by-one in order, suitable when all servers have the same capacity.

### 2. Weighted

```nginx
upstream backend {
    server 10.0.0.1:8080 weight=3;   # Handles 3/6 of requests
    server 10.0.0.2:8080 weight=2;   # Handles 2/6 of requests
    server 10.0.0.3:8080 weight=1;   # Handles 1/6 of requests
}
```

Higher weight = more traffic, suitable when server capacities differ.

### 3. IP Hash

```nginx
upstream backend {
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

**The same client IP always goes to the same server**, ideal for session persistence.

Notes:
- If a server goes down, its IPs are redistributed to other nodes
- If clients access through a proxy, IP hash may not work (Nginx sees the proxy IP)

### 4. Least Connections

```nginx
upstream backend {
    least_conn;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

Sends requests to the server with the fewest active connections, ideal when request processing times vary significantly.

### 5. Random

```nginx
upstream backend {
    random;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
}
```

Supported from Nginx 1.15.7+, picks a random server from the available ones.

### 6. Consistent Hash

```nginx
upstream backend {
    hash $request_uri consistent;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

Hashes based on a custom key (URI, Cookie, etc.). The `consistent` parameter enables consistent hashing, minimizing impact when servers are added or removed.

---

## Server Parameters

```nginx
upstream backend {
    # Basic definition
    server 10.0.0.1:8080;

    # Weight
    server 10.0.0.2:8080 weight=5;

    # Max failures and fail timeout
    server 10.0.0.3:8080 max_fails=3 fail_timeout=30s;

    # Max connections (with queuing)
    server 10.0.0.4:8080 max_conns=100;

    # Slow start (gradually increase traffic after going online)
    server 10.0.0.5:8080 slow_start=30s;

    # Backup (only used when all others are down)
    server 10.0.0.6:8080 backup;

    # Marked as down (no load, no error)
    server 10.0.0.7:8080 down;
}
```

| Parameter | Description | Default |
|-----------|-------------|---------|
| `weight` | Weight | 1 |
| `max_fails` | Max failures before marking as unavailable | 1 |
| `fail_timeout` | Failure timeout duration | 10s |
| `max_conns` | Max concurrent connections | 0 (unlimited) |
| `slow_start` | Slow start duration | 0 (disabled) |
| `backup` | Mark as backup | - |
| `down` | Mark as permanently down | - |

---

## Health Checks

### Passive Health Checks (Built-in)

The open-source version of Nginx only supports passive health checks — determining server health through actual request responses:

```nginx
upstream backend {
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
}
```

After `max_fails` consecutive failures (connection timeout, HTTP 5xx, etc.), no requests are sent to that server for `fail_timeout`.

### Active Health Checks (Nginx Plus or Third-party)

Nginx Plus supports active health checks:

```nginx
upstream backend {
    zone backend 64k;

    server 10.0.0.1:8080;
    server 10.0.0.2:8080;

    health_check interval=5s fails=3 passes=2 uri=/health;
}
```

The open-source version can use the `nginx_upstream_check_module` third-party module for similar functionality.

---

## Retry Mechanism

```nginx
location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_500 http_502 http_503;
    proxy_next_upstream_tries 3;
    proxy_next_upstream_timeout 10s;
}
```

When the upstream server returns specified error codes, Nginx tries the next node.

```nginx
# Non-idempotent requests (POST, PUT) are not retried by default
proxy_next_upstream error timeout;
```

**Note**: Be careful about enabling retries for non-idempotent requests like POST to avoid issues like duplicate orders.

---

## Session Persistence

### Option 1: IP Hash

```nginx
upstream backend {
    ip_hash;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}
```

### Option 2: Cookie Affinity

```nginx
upstream backend {
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    sticky cookie srv_id expires=1h domain=.example.com path=/;
}
```

Requires Nginx Plus or the `nginx-sticky-module` third-party module.

### Option 3: Shared Session (Recommended)

Don't rely on load balancing strategy — store sessions in Redis/Memcached:

```python
# Pseudocode
session_store = RedisSessionStore(host='redis-cluster.example.com')
```

This is the most scalable approach — scaling backend servers is completely transparent to the frontend.

---

## Keep-Alive Connections

Maintaining long-lived connections to the backend when proxying can significantly improve performance:

```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;                       # Max idle connections
    keepalive_requests 100;             # Max requests per connection
    keepalive_timeout 60s;              # Idle timeout
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

**Key points**:
- Must set `proxy_http_version 1.1`
- Must clear the `Connection` header
- `keepalive` limits idle connections, not total connections

---

## Complete Example

```nginx
upstream api_backend {
    least_conn;
    keepalive 32;

    server api1.example.com:8080 weight=5 max_fails=3 fail_timeout=30s;
    server api2.example.com:8080 weight=5 max_fails=3 fail_timeout=30s;
    server api3.example.com:8080 weight=3 max_fails=3 fail_timeout=30s;
    server api4.example.com:8080 backup;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://api_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_next_upstream error timeout http_500;
        proxy_next_upstream_tries 3;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }
}
```

---

## Summary

This article covered Nginx's 6 load balancing algorithms, server parameters, health checks, session persistence, and keep-alive configuration.

---

