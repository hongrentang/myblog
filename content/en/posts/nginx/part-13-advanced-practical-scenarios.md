---
title: "Nginx from Beginner to Pro · Part 13: Advanced Practical Scenarios"
date: 2025-01-04
weight: 1113
draft: false
tags: ["nginx"]
---

## Introduction

This is the final article in the series, collecting common Nginx configuration scenarios from real production environments. Each scenario is a ready-to-use configuration template.

---

## Scenario 1: SPA Single Page Application

Vue/React single page applications require all routes to return `index.html` except for static assets.

```nginx
server {
    listen 80;
    server_name app.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name app.example.com;

    root /var/www/app/dist;

    # SSL
    ssl_certificate /etc/letsencrypt/live/app.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

    # Static assets (hash filenames, cache forever)
    location /assets/ {
        expires 365d;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # All other paths return index.html (SPA routing)
    location / {
        try_files $uri $uri/ /index.html;

        # Don't cache HTML
        add_header Cache-Control "no-cache, must-revalidate";
    }

    # Deny access to hidden files
    location ~ /\. {
        deny all;
        return 404;
    }
}
```

---

## Scenario 2: HTTPS Reverse Proxy

SSL termination reverse proxy — backend services don't need to handle certificates.

```nginx
upstream backend {
    least_conn;
    keepalive 32;
    server 10.0.0.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.0.2:8080 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;
    server_name api.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;

    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;

        proxy_next_upstream error timeout http_500;
        proxy_next_upstream_tries 2;
    }
}
```

---

## Scenario 3: File Upload Service

```nginx
server {
    listen 80;
    server_name upload.example.com;

    client_max_body_size 500m;         # Max upload 500MB
    client_body_buffer_size 2m;        # Above 2MB writes to temp file
    client_body_timeout 300s;          # Upload timeout 5 minutes

    proxy_connect_timeout 60s;
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    location / {
        proxy_pass http://upload_backend:5000;
        proxy_set_header Host $host;
    }
}
```

---

## Scenario 4: WebSocket Proxy

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

upstream ws_backend {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}

server {
    listen 80;
    server_name ws.example.com;

    location /ws {
        proxy_pass http://ws_backend;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

        # WebSocket long connection timeout
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

---

## Scenario 5: WordPress Site

```nginx
upstream php {
    server unix:/var/run/php/php8.3-fpm.sock;
}

server {
    listen 80;
    server_name blog.example.com;

    root /var/www/wordpress;
    index index.php;

    # Serve static assets directly
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|woff|woff2|ttf|svg)$ {
        expires 30d;
        add_header Cache-Control "public, immutable";
        access_log off;
        log_not_found off;
    }

    # WP built-in files
    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        log_not_found off;
        access_log off;
    }

    # Deny hidden files
    location ~ /\. {
        deny all;
    }

    # WordPress routing
    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # PHP handling
    location ~ \.php$ {
        fastcgi_pass php;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
    }

    # Deny PHP execution in certain directories
    location ~* /(wp-content/uploads|wp-content/cache)/.*\.php$ {
        deny all;
    }
}
```

---

## Scenario 6: gRPC Proxy

Nginx 1.13.10+ supports gRPC proxying:

```nginx
upstream grpc_backend {
    server 10.0.0.1:50051;
    server 10.0.0.2:50051;
}

server {
    listen 443 ssl http2;
    server_name grpc.example.com;

    ssl_certificate /etc/letsencrypt/live/grpc.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grpc.example.com/privkey.pem;

    location / {
        grpc_pass grpc://grpc_backend;
        grpc_set_header Host $host;
        grpc_set_header X-Real-IP $remote_addr;

        # gRPC timeouts
        grpc_connect_timeout 5s;
        grpc_read_timeout 3600s;
        grpc_send_timeout 3600s;
    }
}
```

---

## Scenario 7: Multi-Version API Gateway

```nginx
upstream v1_backend {
    server 10.0.0.1:8080;
}

upstream v2_backend {
    server 10.0.0.2:8080;
}

upstream v3_canary {
    server 10.0.0.3:8080 weight=1;
    server 10.0.0.2:8080 weight=99;  # Only 1% traffic to new version
}

server {
    listen 80;
    server_name api.example.com;

    # Version routing
    location /v1/ {
        proxy_pass http://v1_backend/;
    }

    location /v2/ {
        proxy_pass http://v2_backend/;
    }

    location /v3/ {
        proxy_pass http://v3_canary/;
    }

    # No version defaults to latest stable
    location / {
        proxy_pass http://v2_backend/;
    }

    # Canary release: switch version by Header
    location / {
        if ($http_x_api_version = "v3") {
            proxy_pass http://v3_canary/;
            break;
        }
        proxy_pass http://v2_backend/;
    }
}
```

---

## Scenario 8: Download Site Rate Limiting

```nginx
server {
    listen 80;
    server_name download.example.com;

    root /var/www/downloads;

    # Global rate limit
    limit_rate 200k;            # 200KB/s per connection
    limit_rate_after 10m;       # First 10MB unlimited (fast start)

    # Different rate limits by file type
    location ~* \.(zip|tar\.gz)$ {
        limit_rate 500k;
    }

    location ~* \.(iso|dmg)$ {
        limit_rate 2m;          # ISO files 2MB/s
        limit_rate_after 100m;
    }

    # Premium users no limit
    location /vip/ {
        limit_rate 0;           # 0 means unlimited
        valid_referers server_names ~\.example\.com;
        if ($invalid_referer) {
            limit_rate 100k;
        }
    }

    # Enable directory listing
    location / {
        autoindex on;
        autoindex_exact_size off;
        autoindex_localtime on;
    }

    # Limit concurrent connections
    limit_conn per_ip 5;
    limit_conn_status 503;
}
```

---

## Scenario 9: PHP-FPM Optimized Configuration

```nginx
upstream php_backend {
    server unix:/var/run/php/php8.3-fpm.sock;
    server unix:/var/run/php/php8.3-fpm-2.sock;
}

server {
    location ~ \.php$ {
        fastcgi_pass php_backend;
        fastcgi_index index.php;
        include fastcgi_params;

        # PHP-FPM parameters
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PHP_VALUE "max_execution_time=300\nmax_input_time=300";

        # Buffer optimization
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;

        # Timeouts
        fastcgi_connect_timeout 30s;
        fastcgi_read_timeout 300s;
        fastcgi_send_timeout 300s;

        # Cache
        fastcgi_cache my_cache;
        fastcgi_cache_valid 200 302 10m;
        fastcgi_cache_use_stale error timeout updating;
    }
}
```

---

## Scenario 10: CORS Configuration

```nginx
# Generic CORS config (can be placed in snippets/cors.conf)
set $cors_origin "";
if ($http_origin ~* (https?://.*\.example\.com$)) {
    set $cors_origin $http_origin;
}

add_header Access-Control-Allow-Origin $cors_origin always;
add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE, PATCH, OPTIONS" always;
add_header Access-Control-Allow-Headers "Content-Type, Authorization, X-Requested-With" always;
add_header Access-Control-Allow-Credentials "true" always;
add_header Access-Control-Max-Age "86400" always;

# Preflight requests
if ($request_method = OPTIONS) {
    return 204;
}
```

Include it in any location that needs CORS:

```nginx
location /api/ {
    include snippets/cors.conf;
    proxy_pass http://backend;
}
```

---

## Scenario 11: International Multi-Site

```nginx
map $accept_language $lang {
    default en;
    ~^zh zh;
    ~^ja ja;
    ~^ko ko;
}

server {
    listen 80;
    server_name example.com;

    # Redirect based on Accept-Language
    location / {
        if ($lang = zh) {
            rewrite ^ /zh$uri last;
        }
        if ($lang = ja) {
            rewrite ^ /ja$uri last;
        }
    }

    location /en {
        alias /var/www/example/en;
        index index.html;
    }

    location /zh {
        alias /var/www/example/zh;
        index index.html;
    }

    location /ja {
        alias /var/www/example/ja;
        index index.html;
    }
}

# Subdomain approach
server {
    listen 80;
    server_name en.example.com;
    root /var/www/example/en;
}

server {
    listen 80;
    server_name cn.example.com;
    root /var/www/example/zh;
}
```

---

## Scenario 12: Maintenance Page

```nginx
# Method 1: 503 maintenance page
server {
    listen 80;
    server_name example.com;

    # If maintenance flag file exists, show maintenance page
    if (-f $document_root/maintenance.enable) {
        return 503;
    }

    location / {
        proxy_pass http://backend;
    }

    error_page 503 @maintenance;
    location @maintenance {
        root /var/www/errors;
        rewrite ^(.*)$ /maintenance.html break;
    }
}

# Method 2: Whitelist (internal network and test IPs bypass maintenance)
geo $maintenance_bypass {
    default 0;
    192.168.1.0/24 1;
    10.0.0.0/8 1;
    114.114.114.114 1;  # Test IP
}

server {
    if (-f $document_root/maintenance.enable && $maintenance_bypass = 0) {
        return 503;
    }

    # ... normal configuration
}
```

---

## Troubleshooting Tips

Although the series has ended, you'll inevitably encounter new issues in production. Here are some general debugging approaches:

```bash
# 1. Check configuration syntax
nginx -t

# 2. View error log
tail -f /var/log/nginx/error.log

# 3. View access log (last 100 lines)
tail -100 /var/log/nginx/access.log

# 4. Check process status
ps aux | grep nginx

# 5. Test backend connectivity
curl http://127.0.0.1:8080/health

# 6. Check port listening
ss -tlnp | grep nginx

# 7. Check firewall rules
sudo iptables -L -n | grep 80
```

---

## Series Summary

| Part | Topic | Core Content |
|------|-------|-------------|
| 1 | Meet Nginx | Installation, startup, process model, config structure |
| 2 | Basic Configuration | server/location/root/alias, access control |
| 3 | Reverse Proxy | proxy_pass rules, WebSocket |
| 4 | Load Balancing | 6 algorithms, health checks, session persistence |
| 5 | HTTPS Configuration | SSL certificates, TLS optimization, HSTS |
| 6 | Caching | Proxy cache, browser cache, cache strategies |
| 7 | Rewrite Rules | Location priority, rewrite, try_files |
| 8 | Rate Limiting & Security | Connection/request limiting, IP blacklist, DDoS protection |
| 9 | Log Management | Format customization, variables, log analysis, rotation |
| 10 | Performance Optimization | Worker config, kernel parameters, Gzip, SSL |
| 11 | Static & Dynamic Separation | Static resource separation, CDN, version control |
| 12 | High Availability Architecture | Keepalived active-standby, VIP failover |
| 13 | Advanced Practical Scenarios | 12 real-world configuration templates |

From single-server deployment to high-availability clusters, from troubleshooting to performance tuning — these 13 articles cover the full spectrum of Nginx usage. Hope this series helps you use Nginx effectively in your projects.

---

