---
title: "Nginx from Beginner to Pro · Part 8: Rate Limiting & Security"
date: 2025-01-12
weight: 1108
draft: false
tags: ["nginx"]
---

## Why Rate Limiting?

A service without rate limiting is like a bridge without guardrails — when traffic exceeds design capacity:

- Database connection pool exhaustion
- CPU maxed out, response times skyrocket
- Service avalanche: one server fails, traffic shifts to others, triggering cascading failures
- Malicious attacks: brute force, crawlers, DDoS

Nginx's built-in rate limiting controls traffic before it reaches the backend.

---

## 1. Connection Limiting (limit_conn)

Limit concurrent connections per IP:

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn_zone:10m;

    server {
        location / {
            limit_conn conn_zone 10;        # Max 10 concurrent connections per IP
            limit_conn_status 503;           # Return 503 when exceeded
            limit_conn_log_level warn;       # Log level
        }

        location /download/ {
            limit_conn conn_zone 5;          # Stricter limit for downloads
        }
    }
}
```

`$binary_remote_addr` uses less memory than `$remote_addr` (10MB stores ~160K IPs).

### Per-Domain Limiting

```nginx
http {
    limit_conn_zone $server_name zone=site_zone:10m;

    server {
        location / {
            limit_conn site_zone 100;  # Max 100 concurrent connections for the site
        }
    }
}
```

---

## 2. Request Rate Limiting (limit_req)

Limits requests per second, based on the leaky bucket algorithm:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=req_zone:10m rate=1r/s;

    server {
        location / {
            limit_req zone=req_zone;
            limit_req_status 429;             # Return Too Many Requests
            limit_req_log_level warn;
        }
    }
}
```

### Leaky Bucket vs Token Bucket

**Leaky Bucket (default)**:

```nginx
limit_req zone=req_zone burst=5;
```

```
Requests arrive → [Bucket] → Processed at constant rate
Fixed egress rate (1r/s), burst requests queue up to 5, excess is dropped
```

**Token Bucket (with nodelay)**:

```nginx
limit_req zone=req_zone burst=5 nodelay;
```

```
Requests arrive → Take a token → Process immediately
Burst requests with available tokens are handled immediately
```

### Practical Example

```nginx
http {
    # Normal API: 10 per second per IP
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

    # Login endpoint: 5 per minute per IP (brute force protection)
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

    # Crawler limit: by User-Agent
    limit_req_zone $http_user_agent zone=crawler:10m rate=1r/s;

    server {
        location /api/ {
            limit_req zone=api burst=20 nodelay;
        }

        location /login {
            limit_req zone=login burst=3;
            limit_req_status 429;
        }
    }
}
```

Supported rate units: `r/s` (requests per second), `r/m` (requests per minute).

---

## 3. Request Body Size Limits

```nginx
client_max_body_size 10m;
client_body_buffer_size 128k;
client_body_timeout 60s;
```

Typical scenarios:

```nginx
server {
    location /api/ {
        client_max_body_size 1m;
    }

    location /upload/ {
        client_max_body_size 100m;
        client_body_buffer_size 1m;
        client_body_timeout 120s;
    }
}
```

---

## 4. Request Header Limits

```nginx
client_header_buffer_size 1k;
large_client_header_buffers 4 8k;
client_header_timeout 60s;
```

Common oversized header scenarios:
- Large cookies (4K+)
- Overly long custom auth headers

---

## 5. Timeout Settings

```nginx
keepalive_timeout 65;
keepalive_requests 100;
client_body_timeout 60s;
client_header_timeout 60s;
proxy_connect_timeout 30s;
proxy_read_timeout 60s;
proxy_send_timeout 60s;
```

---

## 6. IP Blacklist/Whitelist

### Whitelist Mode

```nginx
geo $whitelist {
    default 0;
    192.168.1.0/24 1;
    10.0.0.0/8 1;
    103.235.46.39 1;  # Partner IP
}

server {
    location /admin {
        if ($whitelist = 0) {
            return 403;
        }
    }
}
```

### Blacklist Mode

```nginx
geo $blocked_ip {
    default 0;
    1.2.3.4 1;
    5.6.7.0/24 1;
}

server {
    if ($blocked_ip) {
        return 444;  # Close connection without response
    }
}
```

---

## 7. Crawler Management

### Rate Limiting Crawlers

```nginx
http {
    limit_req_zone $http_user_agent zone=crawler:10m rate=1r/s;

    server {
        location / {
            limit_req zone=crawler burst=3 nodelay;
        }
    }
}
```

### Blocking Malicious Crawlers

```nginx
if ($http_user_agent ~* (Semrush|AhrefsBot|MJ12bot|DotBot)) {
    return 403;
}
```

---

## 8. DDoS Protection

### Basic Protection

```nginx
http {
    limit_req_zone $binary_remote_addr zone=ddos:10m rate=30r/s;
    limit_conn_zone $binary_remote_addr zone=conn:10m;

    limit_rate 50k;
    limit_rate_after 1m;

    client_body_timeout 10s;
    client_header_timeout 10s;
    keepalive_timeout 10s;

    server {
        listen 80;
        server_name example.com;

        location / {
            limit_req zone=ddos burst=10 nodelay;
            limit_conn conn 5;
            proxy_pass http://backend;
        }
    }
}
```

### Block Non-Standard Requests

```nginx
if ($request_method !~ ^(GET|HEAD|POST)$) {
    return 405;
}

if ($http_user_agent = "") {
    return 403;
}
```

---

## 9. Security Headers

```nginx
server {
    server_tokens off;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';" always;

    limit_except GET HEAD POST {
        deny all;
    }

    location ~ /\. {
        deny all;
        return 404;
    }

    location ~* (\.env|\.git|\.svn|composer\.json|package\.json) {
        deny all;
        return 404;
    }

    location /admin {
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;
        auth_basic "Admin Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }
}
```

---

## Monitoring Rate Limiting

Add rate limiting info to log format:

```nginx
log_format limit '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" '
                'limit_req="$limit_req_status" limit_conn="$limit_conn_status"';

access_log /var/log/nginx/access.log limit;
```

Check rate-limited requests:

```bash
grep "LIMIT" /var/log/nginx/error.log
grep '"limit_req="1"' /var/log/nginx/access.log
```

---

## Summary

Rate limiting and security are the foundation of service stability. This article covered connection limiting, request rate limiting, IP blacklist/whitelist, crawler management, and DDoS protection.

---

