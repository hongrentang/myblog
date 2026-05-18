---
title: "Nginx from Beginner to Pro · Part 1: Meet Nginx"
date: 2025-01-05
weight: 1101
draft: false
tags: ["nginx"]
featured: true
cover:
  image: "/images/nginx-banner.svg"
  caption: "Nginx from Beginner to Pro"
---

## What is Nginx?

Nginx (pronounced "engine-x") is a high-performance HTTP and reverse proxy server, open-sourced in 2004 by Russian programmer Igor Sysoev. It was designed to solve the C10K problem — handling ten thousand concurrent connections simultaneously.

**Key features:**

- **Event-driven architecture**: Unlike Apache's process/thread model, Nginx uses asynchronous non-blocking I/O, handling tens of thousands of concurrent connections with a single process
- **High concurrency**: Easily handles tens of thousands of concurrent connections
- **Low memory footprint**: Under equivalent load, memory usage is far lower than Apache
- **Modular design**: Features are extended through modules, compiled as needed

---

## Installing Nginx

### CentOS / RHEL

```bash
# Add EPEL repository
sudo yum install epel-release -y
# Install
sudo yum install nginx -y
# Start
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Building from Source (Advanced)

```bash
# Install dependencies
sudo apt install build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev openssl libssl-dev

# Download and compile
wget https://nginx.org/download/nginx-1.26.0.tar.gz
tar -zxvf nginx-1.26.0.tar.gz
cd nginx-1.26.0

./configure \
  --prefix=/usr/local/nginx \
  --with-http_ssl_module \
  --with-http_gzip_static_module \
  --with-http_stub_status_module

make && sudo make install
```

---

## Starting and Stopping

```bash
# systemd management (recommended)
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl reload nginx    # Graceful config reload
sudo systemctl restart nginx
sudo systemctl status nginx

# Using the binary directly
sudo nginx                     # Start
sudo nginx -s stop             # Fast shutdown
sudo nginx -s quit             # Graceful shutdown (finishes current requests)
sudo nginx -s reload           # Graceful config reload
sudo nginx -s reopen           # Reopen log files
```

### Verify Installation

Open a browser and visit `http://<server-ip>`. If you see the **Welcome to nginx** page, installation was successful.

---

## Nginx Process Model

```
              Master Process (root)
              /    |    |    \
        Worker  Worker  Worker  Worker  (nobody/nogroup)
```

- **Master process**: Runs as root, responsible for reading config and managing worker processes
- **Worker processes**: Run as a regular user, actually handle requests. Default count equals number of CPU cores
- Each worker handles connections independently, without interfering with others

### Related Configuration

```nginx
worker_processes auto;          # Auto-set to CPU core count
worker_connections 1024;        # Max connections per worker
```

Max concurrent connections ≈ `worker_processes × worker_connections`

---

## Core Concepts

### 1. Virtual Host (Server Block)

Nginx uses `server` blocks to define virtual hosts. A single Nginx instance can host multiple websites.

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example;
}
```

### 2. Context

Nginx configuration is hierarchical — directives at different levels can inherit and override:

```
main                            # Global configuration
├── events                      # Event model configuration
├── http                        # HTTP-related configuration
│   ├── upstream                # Backend server group
│   ├── server                  # Virtual host
│   │   ├── location            # URI matching rules
│   │   └── location
│   └── server
└── stream                      # TCP/UDP proxy
```

### 3. Configuration File Structure

```bash
/etc/nginx/
├── nginx.conf                  # Main configuration file
├── conf.d/                     # Additional config files (included from main config)
│   └── *.conf
├── sites-available/            # Available site configs
├── sites-enabled/              # Enabled site configs (symlinks to sites-available)
├── modules-enabled/            # Module configs
└── modules-available/
```

---

## Quick Diagnostic Commands

```bash
# Check config syntax for errors
nginx -t

# View compile-time parameters
nginx -V

# Check version
nginx -v
```

`nginx -t` is your most-used command — run it every time you modify the config to confirm the syntax is correct before reloading.

---

## Your First Full Configuration

```nginx
# /etc/nginx/nginx.conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;

events {
    worker_connections 1024;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Log format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## Summary

This article introduced what Nginx is, how to install it, its core concepts, and process model.

---

