---
title: "Nginx from Beginner to Pro · Part 7: Rewrite Rules"
date: 2025-01-11
weight: 1107
draft: false
tags: ["nginx"]
---

## Location Matching Rules

`location` determines which configuration block handles a request URI. It's one of Nginx's most core — and most confusing — directives.

### Matching Priority

```nginx
location = /exact {        # 1. Exact match (highest priority)
    ...
}

location ^~ /static/ {     # 2. Priority prefix match
    ...
}

location ~ \.php$ {        # 3. Regex match (case-sensitive)
    ...
}

location ~* \.(jpg)$ {     # 4. Regex match (case-insensitive)
    ...
}

location / {               # 5. General prefix match (lowest priority)
    ...
}
```

**Matching flow**:

```
Request /logo.jpg
1. Match prefix locations — find the longest match
2. If ^~ matches → use it directly, skip regex
3. Otherwise check regex locations in order
4. If regex matches → use that regex location
5. If no regex matches → use the longest prefix match from step 1
```

### Full Priority Example

```nginx
server {
    root /var/www/html;

    location = / {                    # Exact match / → /var/www/html/index.html
        try_files /index.html =404;
    }

    location = /favicon.ico {         # Exact match
        log_not_found off;
        access_log off;
    }

    location ^~ /static/ {            # Priority prefix
        root /var/www;
        expires 30d;
    }

    location ~ \.php$ {               # Regex match PHP
        fastcgi_pass unix:/var/run/php-fpm.sock;
        include fastcgi_params;
    }

    location ~* \.(jpg|png|gif)$ {    # Regex match images
        expires 7d;
    }

    location / {                      # Fallback
        try_files $uri $uri/ /index.php?$query_string;
    }
}
```

### Location Nesting

Nginx doesn't support nested locations, but you can simulate it:

```nginx
# Method 1: Use if in location (not recommended for complex nesting)
location /admin {
    # Rules under /admin
    ...
}

# Method 2: Use include to reuse config
location /api/ {
    include snippets/cors.conf;
    proxy_pass http://api_backend;
}

location /api/v2/ {
    include snippets/cors.conf;
    proxy_pass http://api_v2_backend;
}
```

---

## The rewrite Directive

`rewrite` is used to modify the request URI. Syntax:

```nginx
rewrite regex replacement [flag];
```

### Basic Usage

```nginx
# Rewrite /article/123 to /article?id=123
rewrite ^/article/(\d+)$ /article?id=$1 last;

# Rewrite /page/about to /about.html permanently
rewrite ^/page/(.*)$ /$1.html permanent;
```

### Four Flags

| Flag | Behavior | Use Case |
|------|----------|----------|
| `last` | Stop current location, restart location matching | Internal rewrite |
| `break` | Stop current location, no re-matching | One-time rewrite |
| `redirect` | Return 302 temporary redirect | Temporary redirect |
| `permanent` | Return 301 permanent redirect | Permanent redirect |

### last vs break

```nginx
location /a/ {
    rewrite ^/a/(.*) /b/$1 last;    # → Restart location matching
    # Next line won't execute
    return 200 "from a";
}

location /b/ {
    rewrite ^/b/(.*) /c/$1 break;   # → No re-matching, continue current block
    # Directives after break still execute
    return 200 "from b";
}

location /c/ {
    return 200 "from c";
}
```

The key difference: `last` re-enters location matching, while `break` does not.

---

## The try_files Directive

`try_files` tries files in order, commonly used in SPAs and WordPress:

```nginx
# SPA: if file exists serve it, otherwise return index.html
location / {
    try_files $uri $uri/ /index.html;
}

# With parameter passing
location / {
    try_files $uri $uri/ /index.php?$query_string;
}

# Custom error
location / {
    try_files $uri $uri/ /error.html =404;
}
```

Execution order: try `$uri` (request path), then `$uri/` (as directory), finally fall back to `/index.html`.

---

## The return Directive

```nginx
location /old-path {
    return 301 /new-path;
}

location /gone {
    return 410;  # Resource permanently deleted
}

location /api {
    return 200 "OK";  # Return text directly
}
```

Common HTTP status codes:

| Code | Meaning | Use Case |
|------|---------|----------|
| 301 | Permanent redirect | Domain change, SEO |
| 302 | Temporary redirect | Maintenance |
| 307 | Temporary (preserve method) | POST redirect |
| 308 | Permanent (preserve method) | POST permanent redirect |
| 404 | Not found | Resource doesn't exist |
| 410 | Gone | Resource permanently deleted |
| 444 | Close connection (Nginx-specific) | Reject malicious requests |

---

## Practical Scenarios

### Scenario 1: HTTP to HTTPS

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    return 301 https://$server_name$request_uri;
}
```

### Scenario 2: Force www Prefix

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://www.example.com$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com;
    return 301 https://www.example.com$request_uri;
}

server {
    listen 443 ssl;
    server_name www.example.com;
    # Real site config
}
```

### Scenario 3: Remove index.html

```nginx
if ($request_uri ~ ^/(.*)index\.html$) {
    return 301 /$1;
}
```

### Scenario 4: Mobile Redirect

```nginx
location / {
    if ($http_user_agent ~* "(android|iphone|ipad)") {
        rewrite ^(.*)$ /mobile$1 last;
    }
}

location /mobile {
    root /var/www/mobile;
}
```

### Scenario 5: Maintenance Page

```nginx
location / {
    if (-f $document_root/maintenance.html) {
        return 503;
    }
}

error_page 503 @maintenance;
location @maintenance {
    rewrite ^(.*)$ /maintenance.html break;
}
```

Place a `maintenance.html` in the root directory — all requests redirect to the maintenance page. Delete the file to restore service.

---

## The if Directive Trap

Using `if` inside a location requires caution — **in many cases, if behaves differently than expected**.

### Safe Usage

```nginx
# ✅ Only use return or rewrite ... last
if ($scheme != "https") {
    return 301 https://$server_name$request_uri;
}

if ($request_method = POST) {
    return 405;
}

if ($http_user_agent ~* (curl|wget)) {
    return 403;
}

if ($request_uri ~* "\.(bak|old|swp)$") {
    return 404;
}
```

### Unsafe Usage (Don't Do This)

```nginx
# ❌ Don't use if for try_files work
if (-f $document_root/cache/$uri) {
    rewrite ^ /cache/$uri break;
}

# ❌ Don't use proxy_pass inside if (unpredictable)
if ($uri ~* ^/api/) {
    proxy_pass http://api_backend;
}
```

**Golden rule**: only put `return` or `rewrite ... last` inside `if`.

---

## Debugging Rewrite Rules

```nginx
location /debug {
    # View rewrite process logs
    rewrite_log on;
    return 200 "debug location\n";
}
```

With `rewrite_log` enabled, detailed rewrite process appears in `error_log`:

```nginx
error_log /var/log/nginx/error.log notice;
```

Sample log output:

```
2025/01/15 10:00:00 [notice] 1234#1234: *1 "^/article/(\d+)" matches "/article/42", client: 1.2.3.4
2025/01/15 10:00:00 [notice] 1234#1234: *1 rewritten data: "/article?id=42", args: ""
```

---

## Summary

Location matching and rewrite are the most complex yet most powerful parts of Nginx configuration. Master the priority rules and safe if usage, and you'll write flexible, reliable configurations.

---

