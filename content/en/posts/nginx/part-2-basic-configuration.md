---
title: "Nginx from Beginner to Pro · Part 2: Basic Configuration"
date: 2025-01-06
weight: 1102
draft: false
tags: ["nginx"]
---

## Configuration File Structure Review

Nginx configuration follows a hierarchy of **directives** and **blocks**:

```
directive_name parameter;
block_name { ... }
```

Each directive ends with a semicolon, and blocks are wrapped in curly braces.

---

## Setting Up a Static File Server

### The Simplest Example

```nginx
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static;
    index index.html index.htm;
}
```

Place your HTML files in `/var/www/static` and open `http://static.example.com` in your browser to see the content.

### Full Static Server Configuration

```nginx
server {
    listen 80;
    server_name static.example.com;
    root /var/www/static;
    index index.html;

    # Disable directory listing (security)
    autoindex off;

    # File expiration
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Font files
    location ~* \.(woff|woff2|ttf|otf|eot)$ {
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    # Deny hidden files
    location ~ /\. {
        deny all;
        return 404;
    }
}
```

---

## Configuration Directives Explained

### listen

Listen on a port or Unix socket:

```nginx
listen 80;                          # IPv4
listen [::]:80;                     # IPv6
listen 443 ssl;                     # HTTPS
listen 8080 default_server;         # Default server
listen unix:/var/run/nginx.sock;    # Unix Socket
```

### server_name

Domain matching methods:

```nginx
server_name example.com;                 # Exact match
server_name *.example.com;               # Wildcard
server_name .example.com;                # Matches example.com and *.example.com
server_name ~^www\d\.example\.com$;      # Regex match
server_name "";                          # Matches when no Host header
```

Separate multiple domains with a space: `server_name site1.com site2.com;`

### location

URI matching rules (covered in depth in Part 7):

```nginx
location / {                        # Prefix match
    ...
}

location = /exact {                 # Exact match
    ...
}

location ~ \.php$ {                 # Regex match (case-sensitive)
    ...
}

location ~* \.(jpg|png)$ {         # Regex match (case-insensitive)
    ...
}

location ^~ /static/ {             # Priority prefix match
    ...
}
```

### root vs alias

These two directives are easily confused:

```nginx
# root: appends the full URI to the path
location /images {
    root /var/www;
    # Request /images/logo.png → /var/www/images/logo.png
}

# alias: replaces the matched location portion with the specified path
location /images {
    alias /var/www/uploads;
    # Request /images/logo.png → /var/www/uploads/logo.png
}
```

**Key difference**: `root` does not discard the matched location portion, `alias` does.

---

## Handling Error Pages

```nginx
server {
    error_page 404 /404.html;
    error_page 500 502 503 504 /50x.html;

    location = /404.html {
        root /var/www/errors;
        internal;       # Only accessible through internal redirects
    }

    location = /50x.html {
        root /var/www/errors;
        internal;
    }
}
```

Always use `internal` for custom error pages to prevent direct user access.

---

## Access Control

### IP Whitelist

```nginx
location /admin {
    allow 192.168.1.0/24;
    allow 10.0.0.1;
    deny all;
}
```

### Password Authentication

```bash
# Create password file
sudo htpasswd -c /etc/nginx/.htpasswd admin
```

```nginx
location /admin {
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

### Restricting Request Methods

```nginx
if ($request_method !~ ^(GET|HEAD|POST)$) {
    return 405;
}
```

`limit_except` is the preferred approach:

```nginx
location /api {
    limit_except GET POST {
        deny all;
    }
}
```

---

## Include Directive and Config Splitting

Splitting configuration into multiple files is a good practice:

```nginx
# Main config
include /etc/nginx/conf.d/*.conf;

# Separate upstream definitions
include /etc/nginx/upstreams/*.conf;
```

Common splitting structure:

```
/etc/nginx/
├── nginx.conf                # Global config
├── conf.d/
│   ├── example.com.conf      # One file per site
│   └── api.example.com.conf
├── upstreams/
│   ├── backend.conf          # Backend server groups
│   └── cache.conf
└── snippets/
    ├── ssl.conf              # Common SSL config
    ├── gzip.conf             # Common Gzip config
    └── security.conf         # Security related config
```

---

## Common Variables

Nginx provides many built-in variables for use in configuration:

```nginx
# Using in log format
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent"';

# Checking in location
location / {
    if ($http_user_agent ~* (curl|wget)) {
        return 403;
    }
}
```

**Common variables:**

| Variable | Description |
|----------|-------------|
| `$remote_addr` | Client IP |
| `$remote_user` | Basic auth username |
| `$time_local` | Local time |
| `$request` | Full request line (method + URI + protocol) |
| `$status` | Response status code |
| `$body_bytes_sent` | Response body bytes |
| `$http_referer` | Referer header |
| `$http_user_agent` | User-Agent header |
| `$http_x_forwarded_for` | X-Forwarded-For header |
| `$host` | Requested Host |
| `$uri` | Current URI (may be modified by rewrite) |
| `$request_uri` | Original URI (unmodified) |
| `$scheme` | Protocol (http/https) |
| `$server_name` | Matched server_name |

---

## Debugging Tips

```bash
# Check config syntax
nginx -t

# View all resolved configuration
nginx -T

# Test location matching
# Add inside server block:
location /test {
    return 200 "matched: /test\n";
}
# Then curl http://localhost/test

# View configuration context
nginx -T 2>&1 | grep -A5 "server_name example.com"
```

---

## Summary

In this article we learned the core directives of Nginx basic configuration, including `listen`, `server_name`, `location`, `root/alias`, error pages, access control, and other practical settings.

---

