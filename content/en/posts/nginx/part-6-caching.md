---
title: "Nginx from Beginner to Pro · Part 6: Caching"
date: 2025-01-10
weight: 1106
draft: false
tags: ["nginx"]
---

## Why Caching?

Caching is essentially **trading space for time** — storing processed results and returning them directly on the next request, avoiding repeated computation.

Scenario comparison:

| Scenario | Without Cache | With Cache |
|----------|---------------|------------|
| Homepage load | 200ms + backend rendering | 5ms direct return |
| Hot article | Database query every time | Static HTML return |
| API response | Business logic every time | Direct return during cache period |

---

## Nginx Caching System

Nginx caching operates at two levels:

1. **Proxy Cache**: Caches backend responses at the Nginx layer
2. **Browser Cache**: Controls local browser caching through response headers

---

## 1. Proxy Cache

### Basic Configuration

```nginx
# Define cache path in the http block
http {
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=my_cache:10m
                     max_size=1g inactive=60m use_temp_path=off;

    server {
        location / {
            proxy_pass http://backend;
            proxy_cache my_cache;
            proxy_cache_valid 200 302 10m;
            proxy_cache_valid 404 1m;
            proxy_cache_valid any 1m;
        }
    }
}
```

### Parameter Reference

| Parameter | Description | Example |
|-----------|-------------|---------|
| `levels` | Cache directory hierarchy | `1:2` → `/c/29` |
| `keys_zone` | Shared memory for cache keys | `my_cache:10m` (~80000 keys) |
| `max_size` | Max disk usage for cache | `1g` |
| `inactive` | Cache retention after last access | `60m` |
| `use_temp_path` | Whether to use temp path | `off` (write directly to cache dir) |

### Cache Hit Detection

Add response headers to view cache status:

```nginx
add_header X-Cache-Status $upstream_cache_status;
```

Possible values:
- `HIT` — Cache hit
- `MISS` — Cache miss
- `EXPIRED` — Cache expired
- `STALE` — Expired but stale version used
- `BYPASS` — Manually bypassed cache
- `REVALIDATED` — Validated by upstream server

### Cache Key

The default cache key is determined by the full URL:

```nginx
# View default cache key
proxy_cache_key $scheme$proxy_host$uri$is_args$args;

# Custom cache key (e.g., ignore timestamp in query string)
proxy_cache_key $scheme$proxy_host$uri;
```

### Cache Bypass and Force Refresh

```nginx
location / {
    proxy_pass http://backend;
    proxy_cache my_cache;

    # Don't cache requests with sessionid cookie
    proxy_no_cache $cookie_sessionid;

    # Only cache GET/HEAD
    proxy_cache_methods GET HEAD;

    # Manual cache bypass (specific param to refresh)
    proxy_cache_bypass $arg_nocache;
    proxy_no_cache $arg_nocache;
}
```

Access with `?nocache=1` to bypass the cache and get fresh content.

### Cache Locking

Under high concurrency, multiple requests may miss the cache simultaneously, all hitting the backend:

```nginx
proxy_cache_lock on;
proxy_cache_lock_timeout 5s;
```

The first request fetches from backend, subsequent requests wait. If no response within 5s, subsequent requests pass through.

### Cache Key by Cookie

```nginx
proxy_cache_key $scheme$proxy_host$uri$is_args$args$cookie_user_type;
```

Different user types get different cached versions.

### Clearing Cache

Requires the ngx_cache_purge module or manual deletion:

```bash
# Find cache files
grep -lr "GET /some-path" /var/cache/nginx

# Delete cache directory
rm -rf /var/cache/nginx/*
```

Nginx Plus supports auto-purging via `purger` configuration.

---

## 2. Browser Cache

Browser cache is controlled by Nginx through response headers. It doesn't use storage space but is uncontrollable (users can clear it anytime).

### Expires Header

```nginx
location ~* \.(jpg|jpeg|png|gif|ico)$ {
    expires 30d;
}

location ~* \.(css|js)$ {
    expires 7d;
}

location ~* \.(html)$ {
    expires 1h;
}

location / {
    expires -1;  # No cache
}
```

### Cache-Control Header

`expires` actually sets `Cache-Control` and `Expires` headers under the hood. You can also control them manually:

```nginx
# Static assets: strong cache
location /static/ {
    add_header Cache-Control "public, max-age=2592000, immutable";
}

# HTML pages: conditional cache
location / {
    add_header Cache-Control "no-cache";
}

# API: no cache
location /api/ {
    add_header Cache-Control "no-store, must-revalidate";
}
```

### Common Cache-Control Values

| Directive | Meaning |
|-----------|---------|
| `public` | Any intermediate node can cache |
| `private` | Only browser can cache |
| `no-cache` | Must revalidate before serving from cache |
| `no-store` | Prohibit caching (browser and proxy) |
| `max-age=3600` | Cache valid for 3600 seconds |
| `immutable` | Resource never changes (combined with long max-age) |
| `must-revalidate` | Must revalidate after expiry |

### ETag and Last-Modified

Nginx enables ETag and Last-Modified by default. Browsers send `If-None-Match` and `If-Modified-Since` headers; Nginx returns `304 Not Modified` if the file hasn't changed.

```nginx
# Default behavior, no explicit config needed
# etag on;
# if_modified_since exact;
```

---

## 3. Slice Cache

Enable slice caching for large files, supporting partial content updates and resume downloads:

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=slice_cache:10m;

server {
    location / {
        proxy_cache slice_cache;
        proxy_cache_key $uri$is_args$args;
        slice 1m;  # 1MB per slice

        proxy_cache_valid 200 206 24h;

        # Set range for origin fetch
        proxy_set_header Range $slice_range;
    }
}
```

Use cases: video streaming, large file downloads.

---

## Practical Caching Scenarios

### Scenario 1: WordPress Site Cache

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=wp_cache:10m max_size=1g;

server {
    listen 80;
    server_name blog.example.com;

    location / {
        proxy_pass http://php_backend;
        proxy_cache wp_cache;
        proxy_cache_key $scheme$host$uri;

        # Admin not cached
        proxy_no_cache $cookie_wordpress_logged_in;
        proxy_cache_bypass $cookie_wordpress_logged_in;

        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
    }

    # Static assets served directly by Nginx
    location ~* \.(css|js|jpg|jpeg|png|gif|ico|woff)$ {
        root /var/www/wordpress;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
}
```

### Scenario 2: API Cache

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:50m max_size=2g;

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://api_backend;
        proxy_cache api_cache;
        proxy_cache_methods GET HEAD;
        proxy_cache_key $scheme$host$uri$is_args$args;

        proxy_cache_valid 200 302 5m;
        proxy_cache_valid 404 1m;
        proxy_cache_valid 500 502 503 0s;  # Don't cache errors

        proxy_cache_lock on;
        proxy_cache_lock_timeout 5s;

        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

---

## Caching Strategy Decision Tree

```
Request arrives
├─ Is it a static file (css/js/img)?
│   ├─ Yes → Browser cache (long max-age + immutable)
│   └─ No  → Proxy cache
├─ Does the request need real-time data?
│   ├─ Yes → No cache or very short cache (seconds)
│   └─ No  → Proxy cache
├─ How often does the resource change?
│   ├─ Never → Browser cache + proxy cache
│   ├─ Occasionally → Short proxy cache + browser no-cache
│   └─ Frequently → No cache
```

---

## Summary

Caching is the most direct and effective way to improve performance. This article covered Nginx proxy cache and browser cache configuration in full.

---

