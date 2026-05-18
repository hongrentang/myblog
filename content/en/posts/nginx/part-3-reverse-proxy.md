---
title: "Nginx from Beginner to Pro · Part 3: Reverse Proxy"
date: 2025-01-07
weight: 1103
draft: false
tags: ["nginx"]
---

## What is a Reverse Proxy?

**Forward proxy**: Proxies for the client, helping clients access resources they can't reach directly (e.g., company intranet proxy).

**Reverse proxy**: Proxies for the server — the client doesn't know which backend server ultimately handles the request.

```
Client → Nginx (Reverse Proxy) → Backend Server (Tomcat/Node.js/Python...)
```

Typical use cases for reverse proxy:

- Hide the real address of backend servers
- Unified entry point (one domain for multiple backend services)
- Load balancing
- SSL termination (certificates managed at the Nginx layer)
- Caching and static resource separation

---

## The Simplest Reverse Proxy

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

All requests to `api.example.com` are forwarded to port `8080` on the local machine.

---

## proxy_pass Syntax Details

### With URI vs Without URI

```nginx
# Without URI: forwards the full URI as-is
location /api {
    proxy_pass http://backend;
    # Request /api/users → http://backend/api/users
}

# With URI (trailing /): matched portion is replaced
location /api {
    proxy_pass http://backend/;
    # Request /api/users → http://backend/users
}

location /api {
    proxy_pass http://backend/v2/;
    # Request /api/users → http://backend/v2/users
}
```

**This is one of the easiest places to make a mistake.** Remember one rule:

> Whether `proxy_pass` has a trailing `/` determines if the matched location portion is replaced.

---

## Standard Reverse Proxy Configuration

```nginx
location / {
    proxy_pass http://backend;

    # Set request headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Disable default headers (needed in some environments)
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";

    # Timeout settings
    proxy_connect_timeout 30s;
    proxy_read_timeout 60s;
    proxy_send_timeout 60s;

    # Buffering
    proxy_buffering on;
    proxy_buffer_size 4k;
    proxy_buffers 8 4k;
    proxy_busy_buffers_size 8k;

    # Request body size limit
    client_max_body_size 10m;
}
```

### Configuration Details

| Directive | Description | Default |
|-----------|-------------|---------|
| `proxy_set_header` | Modify forwarded request headers | see below |
| `proxy_connect_timeout` | Backend connection timeout | 60s |
| `proxy_read_timeout` | Backend response read timeout | 60s |
| `proxy_send_timeout` | Backend data send timeout | 60s |
| `proxy_buffering` | Enable buffering | on |
| `client_max_body_size` | Max client request body | 1m |

The default `Host` header value is `$proxy_host` (the backend address). You typically want to change it to `$host` so the backend knows the original domain.

---

## Proxying Different Paths to Different Backends

```nginx
server {
    listen 80;
    server_name example.com;

    # API requests to Java backend
    location /api/ {
        proxy_pass http://java_backend:8080/;
        proxy_set_header Host $host;
    }

    # User center to Python backend
    location /user/ {
        proxy_pass http://python_backend:5000/;
        proxy_set_header Host $host;
    }

    # WebSocket forwarding
    location /ws/ {
        proxy_pass http://websocket_server:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Static files handled directly by Nginx
    location /static/ {
        root /var/www;
        expires 7d;
    }
}
```

---

## WebSocket Proxy

WebSocket requires explicit support for the Upgrade header:

```nginx
location /ws {
    proxy_pass http://ws_backend;

    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;

    # WebSocket long connections need longer timeouts
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

Key points:
- `proxy_http_version 1.1`: WebSocket requires HTTP/1.1
- `Upgrade` and `Connection` headers must be passed through
- Timeouts should be set high, otherwise connections will be dropped

---

## Upstream Server Group

When there are multiple backend instances, use `upstream` to define a group:

```nginx
upstream backend {
    server 10.0.0.1:8080 weight=3;
    server 10.0.0.2:8080 weight=2;
    server 10.0.0.3:8080 backup;     # Backup

    keepalive 32;                      # Keep-alive connections
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

`upstream` is the foundation of load balancing.

---

## Choosing the Backend Protocol

```nginx
# HTTP
proxy_pass http://backend;

# HTTPS
proxy_pass https://backend;

# Unix Socket (better performance)
proxy_pass http://unix:/tmp/backend.sock;

# FastCGI (for PHP)
fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
```

For local communication, Unix sockets are more efficient than TCP, suitable for PHP-FPM and similar scenarios.

---

## Common Troubleshooting

### 502 Bad Gateway

The backend service is not running or the port is wrong.

```bash
# Check if backend is alive
curl http://127.0.0.1:8080

# Check Nginx error log
tail -f /var/log/nginx/error.log
```

### 504 Gateway Timeout

Backend processing took too long, Nginx didn't receive a response within `proxy_read_timeout`.

```nginx
# Increase timeout for slow endpoints
location /slow-api {
    proxy_pass http://backend;
    proxy_read_timeout 300s;
}
```

### 404 Not Found

Usually a URI path mismatch with `proxy_pass` — check for trailing slash.

### Missing Headers

Check if `proxy_set_header` is configured correctly, especially `Host` and `X-Forwarded-*` headers.

---

## Getting the Real Client IP

After passing through a reverse proxy, the backend sees Nginx's IP as the client IP. Forward the real IP with:

```nginx
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Retrieving in backend code:

```python
# Python
real_ip = request.headers.get('X-Real-IP') or request.headers.get('X-Forwarded-For')

# Java
String realIp = request.getHeader("X-Real-IP");
```

---

## Summary

Reverse proxy is one of Nginx's most essential capabilities. This article covered basic configuration, path forwarding rules, WebSocket proxying, and upstream server groups.

---

