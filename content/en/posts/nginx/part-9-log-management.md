---
title: "Nginx from Beginner to Pro · Part 9: Log Management"
date: 2025-01-13
weight: 1109
draft: false
tags: ["nginx"]
---

## Log System

Nginx has two types of logs:

- **Access log**: Records detailed information about each request
- **Error log**: Records Nginx runtime errors and warnings

---

## 1. Access Log

### Default Configuration

```nginx
http {
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;
}
```

### Custom Log Format

```nginx
log_format json escape=json '{'
    '"time_local":"$time_local",'
    '"remote_addr":"$remote_addr",'
    '"remote_user":"$remote_user",'
    '"request":"$request",'
    '"status":$status,'
    '"body_bytes_sent":$body_bytes_sent,'
    '"request_time":$request_time,'
    '"upstream_response_time":"$upstream_response_time",'
    '"http_referer":"$http_referer",'
    '"http_user_agent":"$http_user_agent",'
    '"http_x_forwarded_for":"$http_x_forwarded_for",'
    '"cache_status":"$upstream_cache_status"'
'}';
```

JSON formatted logs are easier to parse with log collection systems (ELK, Loki).

### Conditional Logging

```nginx
# Don't log health check requests
map $uri $loggable {
    /health 0;
    default 1;
}

server {
    access_log /var/log/nginx/access.log main if=$loggable;
}
```

### Per-Domain Logs

```nginx
http {
    log_format main '...';

    server {
        server_name example.com;
        access_log /var/log/nginx/example.com.access.log main;
    }

    server {
        server_name blog.example.com;
        access_log /var/log/nginx/blog.example.com.access.log main;
    }
}
```

### Buffered Writes

Under high concurrency, buffer logs before writing to disk for better performance:

```nginx
access_log /var/log/nginx/access.log main buffer=64k flush=5s;
```

- `buffer=64k`: 64KB buffer, writes when full
- `flush=5s`: writes at most every 5 seconds even if buffer isn't full

---

## 2. Error Log

### Configuration

```nginx
error_log /var/log/nginx/error.log warn;
```

### Log Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| `debug` | Debug info | Troubleshooting config issues |
| `info` | General info | Development |
| `notice` | Important notices | General monitoring |
| `warn` | Warnings | **Recommended for production** |
| `error` | Errors | Record only errors |
| `crit` | Critical | Urgent issues |
| `alert` | Alerts | Immediate action |
| `emerg` | Emergency | System unavailable |

---

## 3. Log Analysis

### Common Analysis Commands

```bash
# Top 10 IPs by request count
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Total page views
wc -l /var/log/nginx/access.log

# Unique visitors
awk '{print $1}' /var/log/nginx/access.log | sort -u | wc -l

# Status code distribution
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# Slowest 10 requests
awk '{print $NF, $0}' /var/log/nginx/access.log | sort -rn | head -10

# Requests > 3 seconds (JSON format)
grep -o '"request_time":[0-9.]*' access.log | cut -d: -f2 | awk '{if($1>3) count++} END{print count " requests > 3s"}'

# Find 5xx errors
awk '$9 ~ /^5/ {print $1, $7, $9}' /var/log/nginx/access.log | head -20
```

### Using goaccess for Visual Analysis

```bash
# Install
sudo apt install goaccess

# Interactive analysis
goaccess /var/log/nginx/access.log --log-format=COMBINED

# Generate HTML report
goaccess /var/log/nginx/access.log --log-format=COMBINED -o report.html
```

---

## 4. Log Rotation (logrotate)

Nginx doesn't have built-in log rotation — it relies on the system's logrotate.

```bash
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                   # Rotate daily
    rotate 30               # Keep 30 days
    dateext                 # Add date to filename
    missingok               # Don't error if log missing
    notifempty              # Don't rotate empty files
    compress                # Compress old logs
    delaycompress           # Delay compression by one day
    sharedscripts           # Share script across all logs
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

Manual test:

```bash
sudo logrotate -f /etc/logrotate.d/nginx
```

**Note**: After rotation, send the `USR1` signal so Nginx reopens the log files, otherwise it continues writing to the renamed files.

---

## 5. Centralized Log Collection

### Filebeat Configuration

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    paths:
      - /var/log/nginx/access.log
    fields:
      type: nginx-access
    json.keys_under_root: true
    json.overwrite_keys: true

output.elasticsearch:
  hosts: ["http://elasticsearch:9200"]
  index: "nginx-access-%{+yyyy.MM.dd}"
```

### Recommended Architecture

```
Nginx → File → Filebeat → Elasticsearch → Kibana
                 ↘ Kafka → Logstash → Elasticsearch
Nginx → File → Promtail → Loki → Grafana
```

For small teams, the Loki + Grafana stack is recommended — simple to deploy and cost-effective.

---

## Summary

Logs are the first source of information for troubleshooting. This article covered format customization, variable system, analysis commands, log rotation, and collection architecture.

---

