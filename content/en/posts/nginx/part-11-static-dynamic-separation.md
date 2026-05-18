---
title: "Nginx from Beginner to Pro · Part 11: Static & Dynamic Separation"
date: 2025-01-02
weight: 1111
draft: false
tags: ["nginx"]
---

## What is Static & Dynamic Separation?

**Static & dynamic separation** means handling dynamic requests (API, page rendering) and static resources (CSS, JS, images, fonts) separately.

```
Static assets (CSS/JS/IMG) → Nginx serves directly (fast)
Dynamic requests (API/pages) → Nginx proxies to backend (business logic)
```

**Why separate?**

- **Reduce backend load**: Static resources don't need app servers
- **Faster response**: Nginx serves static files 10-100x faster than Tomcat/Node.js
- **Cache friendly**: Static assets can have long cache times
- **CDN friendly**: Static assets can be托管ed to CDN

---

## 1. Basic Separation

### Same Domain Separation

```nginx
server {
    listen 80;
    server_name www.example.com;

    root /var/www/example;

    location /static/ {
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /uploads/ {
        expires 7d;
        add_header Cache-Control "public";
    }

    location / {
        proxy_pass http://backend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Separate Domain Separation (Recommended)

```nginx
# Static domain
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static;

    location / {
        expires 30d;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin "*";

        location ~* \.(\w+)\.(css|js)$ {
            expires 365d;
            add_header Cache-Control "public, immutable";
        }
    }
}

# API domain
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://backend:3000;
        proxy_set_header Host $host;
    }
}

# Frontend domain
server {
    listen 80;
    server_name www.example.com;

    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
    }
}
```

Benefits of using a separate domain (`static.example.com`):
- Browsers limit concurrent connections per domain (typically 6-8)
- Static assets load in parallel with page content
- Easy to migrate to CDN

---

## 2. Version Control & Caching Strategy

### Content Hash Strategy

```nginx
location ~* \.[a-f0-9]{8}\.(css|js)$ {
    expires 365d;
    add_header Cache-Control "public, immutable";
}

location ~* \.(css|js)$ {
    expires 7d;
    add_header Cache-Control "public";
}
```

### Version Number Strategy

```nginx
# Reference with version in HTML
# <link rel="stylesheet" href="/static/css/main.css?v=2.3.1">
```

No special Nginx handling needed — browsers re-request when the URL changes.

---

## 3. CDN Integration

### Nginx Origin to CDN

```nginx
upstream cdn_origin {
    server origin-cdn.example.com:443;
    keepalive 32;
}

server {
    listen 80;
    server_name static.example.com;

    proxy_cache_path /var/cache/nginx/cdn levels=1:2 keys_zone=cdn_cache:1g max_size=10g;

    location / {
        proxy_pass https://cdn_origin;
        proxy_cache cdn_cache;
        proxy_cache_valid 200 7d;
        proxy_cache_use_stale error timeout updating;
        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

### Nginx as Origin for CDN

```nginx
server {
    listen 80;
    server_name origin.example.com;

    allow 103.21.244.0/22;
    allow 103.22.200.0/22;
    deny all;

    root /var/www/static;

    location / {
        expires 30d;
        add_header Cache-Control "public";
    }
}
```

---

## 4. Multi-Version Deployment

### A/B Testing by Cookie

```nginx
server {
    listen 80;
    server_name static.example.com;

    location / {
        if ($cookie_version = "v1") {
            root /var/www/static/v1;
        }
        root /var/www/static/v2;
    }
}
```

### Legacy Browser Fallback

```nginx
location / {
    if ($http_user_agent ~* "MSIE 8|MSIE 7") {
        root /var/www/static/legacy;
    }
}
```

---

## 5. Complete Architecture Example

```nginx
upstream api {
    least_conn;
    keepalive 32;
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
}

upstream ssr {
    server 10.0.0.3:3000;
    server 10.0.0.4:3000;
}

server {
    listen 443 ssl http2;
    server_name www.example.com;

    location / {
        proxy_pass http://ssr;
        proxy_set_header Host $host;
    }

    location /api/ {
        proxy_pass http://api/;
        proxy_no_cache 1;
        proxy_cache_bypass 1;
    }

    location /ws {
        proxy_pass http://ssr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 443 ssl http2;
    server_name static.example.com;

    root /var/www/static;

    location / {
        expires 30d;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin "*";

        location ~* \.[a-f0-9]{8,}\.(css|js|jpg|png)$ {
            expires 365d;
            add_header Cache-Control "public, immutable";
        }
    }
}

server {
    listen 80;
    server_name example.com www.example.com static.example.com;
    return 301 https://$server_name$request_uri;
}
```

### Build Artifact Layout

```
/var/www/static/
├── v1.0.0/
│   ├── css/
│   │   ├── main.a1b2c3d4.css
│   │   └── vendor.e5f6g7h8.css
│   ├── js/
│   └── images/
├── v1.1.0/
└── current -> v1.1.0/
```

---

## 6. SSR + Static Separation

Modern frontend frameworks (Next.js/Nuxt) architecture:

```
User Request
    ↓
Nginx (path routing)
    ├─ /_next/* → Static files (high performance)
    └─ Other → Proxy to SSR Server
                        ↓
                SSR Server (Node.js)
                        ↓
                Returns rendered HTML
```

```nginx
# Next.js static & dynamic separation
server {
    listen 80;
    server_name example.com;

    location /_next/static/ {
        root /var/www/next;
        expires 365d;
        add_header Cache-Control "public, immutable";
    }

    location /static/ {
        root /var/www/next/public;
        expires 30d;
    }

    location / {
        proxy_pass http://nextjs:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

---

## Summary

Static & dynamic separation is a core architectural pattern for improving website performance. This article covered same-domain and separate-domain separation, version control, CDN integration, and complete architecture configuration.

---

