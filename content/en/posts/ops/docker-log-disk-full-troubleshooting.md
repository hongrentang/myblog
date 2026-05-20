---
title: "Docker Logs Filled Up the Disk — From Alert to Root Cause Fix"
date: 2026-05-19
weight: 100060
slug: "docker-log-disk-full-troubleshooting"
tags: ["docker", "linux", "troubleshooting"]
categories: ["运维"]
description: "Complete guide to troubleshooting Docker container logs filling up the disk — from emergency cleanup to long-term prevention"
keywords: "docker log cleanup, docker disk full, json.log cleanup, docker log rotation, daemon.json config"
draft: false
featured: true
cover:
  image: "/images/docker-log-disk-full-banner.svg"
  caption: "Docker Log Disk Full Troubleshooting"
---

# Docker Logs Filled Up the Disk — From Alert to Root Cause Fix

## 1. Symptoms

Disk alert fired — 95% usage.

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G   47G  3.0G  94% /
```

Root partition is nearly full. If left unchecked, databases will fail to write, logs won't flush, even SSH might refuse new connections.

## 2. Investigation

### 2.1 Find the Large Directory

```bash
# Drill down from root — find what's eating disk space
du -sh /* | sort -rh | head -5
```

```
32G   /var
8.5G  /usr
7.2G  /home
```

`/var` is 32G. Dig deeper.

```bash
# Check /var subdirectories
du -sh /var/* | sort -rh | head -5
```

```
30G   /var/lib/docker
```

Docker owns 30G.

### 2.2 Find the Container Logs

```bash
# Container logs live in /var/lib/docker/containers/<id>/*.log
du -sh /var/lib/docker/containers/*/*.log | sort -rh | head -5
```

```
8.2G  /var/lib/docker/containers/abc123/abc123-json.log
5.1G  /var/lib/docker/containers/def456/def456-json.log
3.8G  /var/lib/docker/containers/ghi789/ghi789-json.log
```

**Key finding**: several containers have GB-sized `json.log` files. Docker's default logging driver `json-file` **does NOT rotate logs** — they grow indefinitely until the disk fills up.

### 2.3 Check Log Contents (Optional)

```bash
# See what the container is spamming
tail -50 /var/lib/docker/containers/abc123/abc123-json.log | jq '.log' | head -10
```

```
"2026-05-19 14:00:01 ERROR Failed to connect to database..."
"2026-05-19 14:00:02 ERROR Retrying connection..."
"2026-05-19 14:00:03 ERROR Failed to connect to database..."
```

The app is retrying a database connection in a tight loop, writing ERROR lines every second. That's how the log ballooned from MB to GB.

⚠️ **Pitfall warning**: **Do NOT run `rm *.log`**. The file is still held open by the Docker process — `rm` only removes the filename, the process keeps writing to the same inode, so disk space won't be released. See Solution A below.

## 3. Root Cause

| Layer | Cause |
|-------|-------|
| Direct cause | App continuously writes ERROR logs |
| Surface cause | Docker json-file driver doesn't rotate by default |
| Root cause | No log size limit configured, no log collection pipeline |

Bottom line: **Docker won't manage log size for you. If you don't configure it, it will blow up.**

## 4. Solutions

### Option A: Emergency Cleanup (Stop the Bleeding)

```bash
# 1. Check current log file size
ls -lh /var/lib/docker/containers/abc123/abc123-json.log

# 2. Truncate the file (DO NOT delete!)
truncate -s 0 /var/lib/docker/containers/abc123/abc123-json.log

# 3. Verify disk space is released
df -h /
```

✅ **Success criteria**: `df -h` shows usage dropped.

```bash
# One-liner to truncate all container logs
truncate -s 0 /var/lib/docker/containers/*/*.log
```

**Why truncate instead of rm**:

| Action | Result |
|--------|--------|
| `rm *.log` | ❌ Doesn't free space (process holds fd) |
| `truncate -s 0 file` | ✅ Frees immediately, no container restart |
| `cat /dev/null > file` | ✅ Same as truncate |

### Option B: Per-Container Log Limit

Set limits when starting a container:

```bash
# Limit log file to 100M, keep 3 files max
docker run -d \
  --log-opt max-size=100m \
  --log-opt max-file=3 \
  --name my-app \
  nginx:latest
```

✅ **Verify**:

```bash
# Check container log config
docker inspect my-app --format '{{.HostConfig.LogConfig}}'
```

Expected output includes `max-size:100m max-file:3`.

### Option C: Global Config (Recommended, Set-and-Forget)

```bash
# 1. Edit Docker daemon config
vim /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

```bash
# 2. Reload config (doesn't interrupt running containers)
systemctl reload docker

# 3. Verify config took effect
docker info | grep -A 5 "Logging Driver"
```

```
 Logging Driver: json-file
 Log:
  max-file: 3
  max-size: 100m
```

⚠️ **Pitfall warning**: `systemctl reload docker` **only affects new containers**. Existing containers keep their old config and need to be recreated:

```bash
# Existing containers must be recreated
docker-compose down && docker-compose up -d
```

### Option D: Switch to local Driver (Production Recommended)

Docker's `local` driver has built-in rotation and better performance than `json-file`. It stores logs in binary format:

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
```

**Option Comparison**:

| Option | Scenario | Scope | Restart Needed |
|--------|----------|-------|----------------|
| A. Emergency truncate | Disk nearly full | One-time | No |
| B. Per-container limit | Dev/test | Single container | Recreate needed |
| C. Global json-file | Controlled env | Global | New containers only |
| D. Global local | Production | Global | New containers only |

## 5. Verification

```bash
# 1. Confirm logging driver config
docker info --format '{{.LoggingDriver}}'
# Expected: json-file or local

# 2. Check log files aren't growing
ls -lh /var/lib/docker/containers/*/*.log

# 3. Start a test container and verify rotation
docker run -d --name log-test alpine sh -c "while true; do echo 'test'; done"
sleep 5
ls -lh /var/lib/docker/containers/$(docker ps -q --filter name=log-test)/*.log
docker rm -f log-test
```

✅ **Final verification**:
- `docker info` shows expected Logging Driver and options
- New container logs don't exceed 100M
- Disk usage stays below 80%

## 6. Long-Term Prevention

```bash
# 1. Monitor: alert when disk usage > 80%
# Prometheus: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.2

# 2. Weekly check: find container logs over 500M
find /var/lib/docker/containers/ -name "*.log" -size +500M -exec ls -lh {} \;

# 3. Production: ship logs to external system (ELK / Loki)
# Don't keep verbose logs locally — collect and truncate
```

**Maintenance checklist**:
- [ ] New services must specify `--log-opt max-size`
- [ ] Quarterly check `daemon.json` hasn't been overwritten (some management tools rewrite it)
- [ ] Ensure log collection system has its own disk alert (ELK can blow up too)
