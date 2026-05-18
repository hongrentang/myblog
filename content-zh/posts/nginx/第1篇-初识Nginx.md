---
title: "Nginx 从入门到精通 · 第1篇：初识 Nginx"
date: 2025-01-05
weight: 1101
draft: false
tags: ["nginx"]
featured: true
cover:
  image: "/images/nginx-banner.svg"
  caption: "Nginx 从入门到精通"
---

## 什么是 Nginx？

Nginx（发音 "engine-x"）是一个高性能的 HTTP 和反向代理服务器，由俄罗斯程序员 Igor Sysoev 于 2004 年开源。它的设计目标是解决 C10K 问题——即同时处理一万个并发连接。

**核心特点：**

- **事件驱动架构**：不同于 Apache 的进程/线程模型，Nginx 使用异步非阻塞 I/O，单进程可处理数万并发
- **高并发能力**：轻松支撑数万并发连接
- **低内存消耗**：同等负载下，内存占用远低于 Apache
- **模块化设计**：功能通过模块扩展，可按需编译

---

## 安装 Nginx

### CentOS / RHEL

```bash
# 添加 EPEL 源
sudo yum install epel-release -y
# 安装
sudo yum install nginx -y
# 启动
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

### 源码编译（进阶）

```bash
# 安装依赖
sudo apt install build-essential libpcre3 libpcre3-dev zlib1g zlib1g-dev openssl libssl-dev

# 下载并编译
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

## 启动与停止

```bash
# systemd 管理（推荐）
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl reload nginx    # 平滑重载配置
sudo systemctl restart nginx
sudo systemctl status nginx

# 直接使用二进制
sudo nginx                     # 启动
sudo nginx -s stop             # 快速停止
sudo nginx -s quit             # 优雅停止（处理完当前请求）
sudo nginx -s reload           # 平滑重载配置
sudo nginx -s reopen           # 重新打开日志文件
```

### 验证安装

浏览器访问 `http://服务器IP`，看到 **Welcome to nginx** 页面即成功。

---

## Nginx 进程模型

```
              Master Process (root)
              /    |    |    \
        Worker  Worker  Worker  Worker  (nobody/nogroup)
```

- **Master 进程**：以 root 运行，负责读取配置、管理 Worker 进程
- **Worker 进程**：以普通用户运行，实际处理请求。默认数量等于 CPU 核心数
- 每个 Worker 独立处理连接，互不干扰

### 相关配置

```nginx
worker_processes auto;          # 自动设置为 CPU 核心数
worker_connections 1024;        # 每个 Worker 最大连接数
```

最大并发数 ≈ `worker_processes × worker_connections`

---

## 核心概念

### 1. 虚拟主机（Server Block）

Nginx 通过 `server` 块来定义虚拟主机，一个 Nginx 可以托管多个网站。

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example;
}
```

### 2. 上下文（Context）

Nginx 的配置是分层的，不同层级的配置可以继承和覆盖：

```
main                            # 全局配置
├── events                      # 事件模型配置
├── http                        # HTTP 相关配置
│   ├── upstream                # 后端服务器组
│   ├── server                  # 虚拟主机
│   │   ├── location            # URI 匹配规则
│   │   └── location
│   └── server
└── stream                      # TCP/UDP 代理
```

### 3. 配置文件目录结构

```bash
/etc/nginx/
├── nginx.conf                  # 主配置文件
├── conf.d/                     # 其他配置文件（被主配置 include）
│   └── *.conf
├── sites-available/            # 站点可用配置
├── sites-enabled/              # 站点启用配置（软链接到 sites-available）
├── modules-enabled/            # 模块配置
└── modules-available/
```

---

## 快速检查命令

```bash
# 检查配置语法是否错误
nginx -t

# 查看编译参数
nginx -V

# 查看版本
nginx -v
```

`nginx -t` 是你最常用的命令——每次修改配置后都跑一遍，确认语法无误再 reload。

---

## 第一段完整配置

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

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## 小结

这一篇介绍了 Nginx 是什么、如何安装、核心概念和进程模型。

---

