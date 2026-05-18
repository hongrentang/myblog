---
title: "Nginx from Beginner to Pro · Part 10: Performance Optimization"
date: 2025-01-01
weight: 1110
draft: false
tags: ["nginx"]
---

## Performance Metrics

Before optimizing, establish the **benchmarks**:

| Metric | Description | Target |
|--------|-------------|--------|
| QPS | Queries per second | Higher is better |
| Response time | Request processing time | < 100ms |
| Concurrent connections | Simultaneous connections handled | Meet target |
| Memory usage | Process memory | < 50MB per core |
| CPU usage | Core utilization | Efficient, not idle |

---

## 1. Worker Process Optimization

### Process Count and Connections

```nginx
# Set to CPU core count, auto detects
worker_processes auto;

# Bind workers to specific CPUs (avoid process migration)
worker_cpu_affinity auto;

# Max connections per worker
worker_connections 10240;

# Max file descriptors (system limit must be adjusted too)
worker_rlimit_nofile 65535;
```

**Formula**:

```
Max concurrent connections = worker_processes × worker_connections
Max concurrent clients = Max concurrent connections / 2
```

### Event Model

```nginx
events {
    use epoll;                      # Most efficient on Linux
    worker_connections 10240;
    multi_accept on;                # Accept all new connections at once
    accept_mutex on;                # Prevent thundering herd
}
```

| Model | System | Description |
|-------|--------|-------------|
| `epoll` | Linux 2.6+ | Default, most efficient |
| `kqueue` | FreeBSD/macOS | macOS default |
| `select` | Universal | Worst, fallback |

---

## 2. OS-Level Optimization

### File Descriptor Limits

```bash
ulimit -n
ulimit -n 65535

# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

### TCP Kernel Parameters

```bash
# /etc/sysctl.conf

# Port range (TIME_WAIT port reuse)
net.ipv4.ip_local_port_range = 1024 65535

# TIME_WAIT reuse and fast recycling
net.ipv4.tcp_tw_reuse = 1

# Reduce FIN-WAIT-2 timeout
net.ipv4.tcp_fin_timeout = 30

# Max SYN half-open connections (SYN Flood protection)
net.ipv4.tcp_max_syn_backlog = 65536

# Allow more TIME_WAIT sockets
net.ipv4.tcp_max_tw_buckets = 2000000

# TCP read/write buffers
net.core.rmem_default = 65536
net.core.wmem_default = 65536
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# SOMAXCONN (listen queue length)
net.core.somaxconn = 65535

# Apply
sudo sysctl -p
```

---

## 3. Proxy Performance Optimization

### Enable Keepalive

```nginx
upstream backend {
    server 10.0.0.1:8080;
    keepalive 32;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

### Proxy Buffering

```nginx
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 16k;
proxy_busy_buffers_size 32k;
proxy_temp_file_write_size 32k;

location /events {
    proxy_buffering off;
    proxy_cache off;
}
```

### Reduce Data Copying

```nginx
# Zero-copy: data from disk to NIC without passing through user space
sendfile on;

# Optimize sending (merge small packets)
tcp_nopush on;

# Reduce latency (send small packets immediately)
tcp_nodelay on;
```

---

## 4. Static Resource Optimization

### Gzip Compression

```nginx
gzip on;
gzip_min_length 1000;
gzip_comp_level 3;
gzip_vary on;
gzip_proxied any;
gzip_types
    text/plain
    text/css
    text/xml
    text/javascript
    application/json
    application/javascript
    application/xml+rss
    image/svg+xml;
gzip_disable "msie6";
```

**Compression level reference**:

| Level | CPU Cost | Ratio | Suggestion |
|-------|----------|-------|------------|
| 1 | Low | Poor | Not recommended |
| 2-3 | Low | Good | **Recommended, best value** |
| 4-6 | Medium | Better | If CPU is plentiful |
| 7-9 | High | Best | Not recommended, diminishing returns |

### File Cache

```nginx
open_file_cache max=10000 inactive=30s;
open_file_cache_valid 60s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

### Hotlink Protection

```nginx
location ~* \.(jpg|jpeg|png|gif)$ {
    valid_referers none blocked
        ~\.google\.com
        server_names
        ~\.example\.com;

    if ($invalid_referer) {
        return 403;
    }
}
```

---

## 5. SSL Performance Optimization

```nginx
server {
    listen 443 ssl http2;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets on;

    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers on;
    ssl_ecdh_curve prime256v1:secp384r1;

    ssl_stapling on;
    ssl_stapling_verify on;
}
```

**SSL Optimization Impact**:

| Optimization | Improvement |
|--------------|-------------|
| Session cache | Reduces 90% SSL handshake time |
| HTTP/2 | Multiplexing, fewer connections |
| OCSP Stapling | No extra browser verification |
| TLS 1.3 | 1-RTT handshake, 2x faster than TLS 1.2 |

---

## 6. Log Optimization

```nginx
error_log /var/log/nginx/error.log warn;
access_log /var/log/nginx/access.log main buffer=64k flush=5s;

location /health {
    access_log off;
}
```

---

## 7. Performance Testing

### Using ab (Apache Bench)

```bash
sudo apt install apache2-utils

ab -n 1000 -c 100 https://example.com/
ab -n 1000 -c 100 -k https://example.com/
ab -n 5000 -c 200 https://example.com/api/users
```

### Using wrk

```bash
sudo apt install wrk

wrk -t4 -c200 -d30s https://example.com/
wrk -t4 -c200 -d30s -H "Authorization: Bearer xxx" https://example.com/api
```

### Reading Results

```
Requests per second:    15234.45 [#/sec] (mean)   # QPS
Time per request:       13.127 [ms] (mean)        # Average response time
Transfer rate:          28750.56 [Kbytes/sec]     # Throughput
```

---

## Summary

Performance optimization is a systematic engineering effort. This article covered Worker configuration, kernel parameters, proxy buffering, SSL optimization, and load testing.

---

