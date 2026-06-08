---
title: "File Descriptor Exhaustion — When Too Many Open Files Brought Down the Entire Platform"
date: 2026-06-09
weight: 100430
slug: "file-descriptor-exhaustion-outage"
tags: ["linux", "troubleshooting", "system", "fd", "performance"]
categories: ["Troubleshooting"]
description: "A file descriptor exhaustion incident — how an application socket leak and insufficient ulimit caused all services on a Linux server to fail with 'too many open files', taking down a microservices platform"
keywords: "file descriptor exhaustion, too many open files, ulimit nofile, linux fd leak, socket leak troubleshooting, emfile error"
draft: false
featured: true
cover:
  image: ""
  caption: "File Descriptor Exhaustion — Troubleshooting"
---

# File Descriptor Exhaustion — When Too Many Open Files Brought Down the Entire Platform

## Common Search Queries

- "too many open files" linux
- emfile error
- ulimit nofile troubleshooting
- socket CLOSE_WAIT leak
- file descriptor exhaustion fix
- /proc/sys/fs/file-nr monitoring
- systemd LimitNOFILE
- how to check open file descriptors on linux

---

## The Incident

**Environment**: CentOS 7, 16 vCPU, 64GB RAM, 12 microservices running as systemd units on a bare-metal server.

**Time**: 2:30 PM, right after a routine deployment of a new version of `order-service` (Java).

**Symptoms**:

All 12 microservices on the node started failing simultaneously within minutes of the deploy.

```text
# order-service log
java.io.IOException: Too many open files
    at java.base/sun.nio.ch.SocketChannelImpl.checkConnect(Native Method)

# payment-service log
Caused by: java.net.SocketException: Too many open files

# notification-service log
java.io.FileNotFoundException: /var/log/notification/app.log (Too many open files)
```

Even SSH became painful:

```text
$ ssh production-node-01
-bash: fork: retry: Resource temporarily unavailable
ssh_exchange_identification: Connection closed by remote host
```

And when you did manage to log in:

```text
$ sudo tail -f /var/log/messages
Jun  9 14:32:15 production-node-01 systemd-logind: Failed to start user service 'user-1000.slice': Too many open files
Jun  9 14:32:15 production-node-01 systemd-logind: Failed to create session: Too many open files
```

The node was essentially dead. No new processes could start. Existing processes could not open new files or connections. The platform ground to a halt.

---

## Background — Linux File Descriptor Basics

Before diving into the investigation, let's understand what file descriptors are and why they matter.

### What is a file descriptor?

A **file descriptor (FD)** is a non-negative integer that the operating system uses to identify an open file, socket, pipe, or other I/O resource. Every time a process opens a file, creates a network connection, or establishes a pipe, the kernel allocates an FD.

### Per-process limits: `ulimit -n`

Each process has a limit on how many FDs it can open. This is controlled by `ulimit -n` (nofile):

- **Soft limit**: The effective limit. A process can raise its soft limit up to the hard limit.
- **Hard limit**: The absolute ceiling. Only `root` can raise the hard limit.

```bash
# Check current limits
ulimit -Sn   # soft limit
ulimit -Hn   # hard limit
```

### System-wide limits

The kernel imposes global limits too:

- **`fs.file-max`**: The maximum number of open file descriptors system-wide.
- **`fs.nr_open`**: The per-process hard limit ceiling (usually 1048576).

```bash
# System-wide FD stats
cat /proc/sys/fs/file-max        # 65535 (default on many systems)
cat /proc/sys/fs/file-nr         # 6336  0   65535  (allocated, free, max)
```

The `file-nr` output shows three numbers: `allocated | free | max` — the number of file descriptors currently allocated (used), the number of free (unused) allocated FDs, and the system-wide maximum (file-max).

### How FDs are consumed

File descriptors are used by:

| Resource | Typical FD usage |
|----------|-----------------|
| Open files (logs, configs, data) | 1 FD per file |
| Network sockets | 1 FD per connection |
| Pipes | 2 FDs per pipe |
| epoll instances | 1 FD per instance |
| EventFd / TimerFd | 1 FD each |

In a microservices environment, the biggest FD consumer is almost always **network sockets** — each HTTP request, database connection, cache connection, and RPC call consumes an FD.

---

## Investigation

### Step 1: Check system-wide FD count

First, let's see how many file descriptors are open across the entire system:

```bash
$ cat /proc/sys/fs/file-nr
65280  0  65535
```

**Alarm bells**. 65,280 FDs allocated out of 65,535 max. The system is **99.6% full**. We are 255 FDs away from total exhaustion.

### Step 2: Find which process is consuming FDs

```bash
$ for pid in /proc/[0-9]*; do
    echo "$(basename $pid) $(ls "$pid"/fd 2>/dev/null | wc -l)"
  done | sort -k2 -rn | head -10
```

Output:

```
28765 1024
31233 887
30122 654
28901 432
...
```

Process 28765 has hit its per-process limit: **1024**. That's the default `ulimit -n` on CentOS 7.

### Step 3: Check the offending process

```bash
$ ps aux | grep 28765
java     28765  12.3  8.1  8.2g 5.4g ?  Ssl  14:25  4:32 /usr/bin/java -jar order-service.jar
```

It's the freshly deployed `order-service`.

```bash
$ cat /proc/28765/limits | grep "open files"
Max open files            1024                 4096                 files
```

Soft limit: 1024. Hard limit: 4096. The process hit its soft limit.

### Step 4: What kind of FDs are open?

```bash
$ ls -la /proc/28765/fd | head -20
total 0
dr-x------ 2 java java  ... .
dr-x------ 2 java java  ... ..
lrwxrwxrwx 1 java java  ... 0 -> /dev/null
lrwxrwxrwx 1 java java  ... 1 -> /dev/null
lrwxrwxrwx 1 java java  ... 10 -> socket:[1897201]
lrwxrwxrwx 1 java java  ... 11 -> socket:[1897202]
lrwxrwxrwx 1 java java  ... 12 -> socket:[1897203]
lrwxrwxrwx 1 java java  ... 13 -> socket:[1897204]
...
```

Almost all FDs are **sockets**. Let's check with `lsof`:

```bash
$ lsof -p 28765 | head -30 | awk '{print $5}' | sort | uniq -c | sort -rn
   1024 IPv4
```

Wait — all 1024 entries for this process are sockets. Not a single regular file FD.

### Step 5: Check socket states

```bash
$ lsof -p 28765 | grep -c CLOSE_WAIT
8432
```

**8,432 sockets in `CLOSE_WAIT` state**. This is the smoking gun.

> **CLOSE_WAIT** means the remote end has closed the connection, but the local application (our Java code) has not called `close()` on the socket. The socket is half-closed and will leak indefinitely if the application doesn't clean it up.

### Step 6: Check systemd service configuration

```bash
$ systemctl show order-service | grep -i limitnofile
LimitNOFILE=
LimitNOFILESoft=
```

Both are empty — meaning the service inherits the systemd default, which is typically **1024** on CentOS 7.

### Step 7: Check system-wide configuration

```bash
$ sysctl fs.file-max
fs.file-max = 65535

$ ulimit -n
1024
```

The default `ulimit -n` for user processes is 1024. The system-wide `fs.file-max` of 65535 could accommodate more, but the per-process limit is the bottleneck here — and the total system FD count is also nearly exhausted.

---

## Root Cause

The investigation identified three contributing factors that aligned to create a perfect outage:

### 1. Application-level socket leak (primary cause)

The newly deployed `order-service` (Java) used an HTTP connection pool that was **not configured with a maximum connection limit**. Worse, it did not properly close idle connections:

```java
// Buggy code — leaky connection pool
public class OrderServiceClient {
    private CloseableHttpClient httpClient = HttpClients.createDefault();

    public Response callExternalApi(Request req) {
        // Default HttpClient has no connection limit
        // Idle connections are never evicted
        // No connection timeout or stale check
        HttpGet httpGet = new HttpGet("https://external-api.example.com/orders");
        try {
            return httpClient.execute(httpGet, response -> {
                // Process response but never close the connection explicitly
                return new Response(response.getEntity().getContent());
            });
        } catch (IOException e) {
            log.error("API call failed", e);
            // The connection is NOT closed in the error path either
            throw new RuntimeException(e);
        }
    }
}
```

Key problems:
- `HttpClients.createDefault()` creates a pool with **default max connections per route = 2 and max total = 20**, but the upstream API server was configured with keep-alive, causing connections to be held open.
- The code path that processes responses opens an `InputStream` but never calls `close()`.
- The error path also leaks — on exception, the connection is returned to the pool but the underlying socket may be in a broken state.
- No idle connection eviction was configured (`evictExpiredConnections()` / `evictIdleConnections()`).
- No connection validation (`setValidateAfterInactivity()`).

Over time, as HTTP calls accumulated, the connection pool created new connections while never releasing old ones, leading to a massive socket leak.

### 2. Low per-process FD limit (contributing factor)

The default `ulimit -n` of **1024** on CentOS 7 is woefully inadequate for a Java microservice handling hundreds of requests per second. The `order-service` service hit this limit within approximately 7 minutes of startup.

```bash
# Default on CentOS 7 — too low for modern services
* soft nofile 1024
* hard nofile 4096
```

### 3. systemd inherit default (contributing factor)

The systemd unit file for `order-service` did not specify `LimitNOFILE`, so it inherited the systemd default.

```ini
# Missing from order-service.service
[Service]
# LimitNOFILE=65536  <-- this was not set
```

### The cascade

1. `order-service` starts leaking sockets from the moment it is deployed.
2. Within ~7 minutes, it hits its per-process limit of 1024 FDs and can no longer serve requests.
3. The service stops responding to health checks, degrading all downstream callers.
4. Other services that depend on `order-service` accumulate their own backlogs — retries, queued requests, and connection churn consume FDs rapidly.
5. The node-level FD count approaches `fs.file-max` (65535).
6. `systemd-logind` cannot open FDs for new SSH sessions → logins fail.
7. Existing processes cannot create new threads (needs an FD for `fork()`), cannot write to logs, cannot accept new connections.
8. **Complete platform outage on this node.**

---

## Resolution

### Emergency fix (immediate — 5 minutes)

Stop the leak. Kill the offending process:

```bash
# Identify and kill the leaking service
$ systemctl stop order-service
$ kill -9 28765   # if systemctl doesn't work due to FD exhaustion
```

The moment the process died, all its FDs were reclaimed by the kernel.

```bash
$ cat /proc/sys/fs/file-nr
823  0  65535
```

FD usage dropped from 65,280 to 823. **All other services recovered immediately** — they could open files, accept connections, and write logs again. SSH access returned to normal.

### Short-term fix (30 minutes)

Increase limits to prevent recurrence before the next deploy:

**Per-process via systemd:**

```bash
$ systemctl edit order-service
```

Add:

```ini
[Service]
LimitNOFILE=65536
LimitNOFILESoft=65536
```

Then:

```bash
$ systemctl daemon-reload
$ systemctl restart order-service
```

Verify:

```bash
$ cat /proc/$(pidof order-service)/limits | grep "open files"
Max open files            65536                65536                files
```

**System-wide limits:**

```bash
# Increase kernel max
$ echo "fs.file-max = 200000" >> /etc/sysctl.conf
$ sysctl -p

# Increase per-user limits
$ echo "* soft nofile 65536" >> /etc/security/limits.conf
$ echo "* hard nofile 65536" >> /etc/security/limits.conf
```

### Permanent fix (code change)

The core issue was the Java HTTP connection pool leak. Fix the code:

```java
// Fixed code — proper connection pool management
public class OrderServiceClient {
    private final CloseableHttpClient httpClient;

    public OrderServiceClient() {
        // Use connection pool manager with proper limits
        PoolingHttpClientConnectionManager poolManager =
            new PoolingHttpClientConnectionManager();
        poolManager.setMaxTotal(200);
        poolManager.setDefaultMaxPerRoute(50);
        // Close idle connections after 30 seconds
        poolManager.setValidateAfterInactivity(5000);

        this.httpClient = HttpClients.custom()
            .setConnectionManager(poolManager)
            .setConnectionTimeToLive(30, TimeUnit.SECONDS)
            .evictExpiredConnections()
            .evictIdleConnections(30, TimeUnit.SECONDS)
            .build();
    }

    public Response callExternalApi(Request req) {
        HttpGet httpGet = new HttpGet("https://external-api.example.com/orders");
        try (CloseableHttpResponse response = httpClient.execute(httpGet)) {
            // try-with-resources ensures the response is always closed
            return new Response(EntityUtils.toByteArray(response.getEntity()));
        } catch (IOException e) {
            log.error("API call failed", e);
            throw new RuntimeException(e);
        }
    }
}
```

Key fixes:
- `PoolingHttpClientConnectionManager` with explicit max connections.
- `evictExpiredConnections()` and `evictIdleConnections(30, TimeUnit.SECONDS)` — background eviction thread.
- `setValidateAfterInactivity(5000)` — validate connections before use.
- `try-with-resources` — ensures `close()` is always called, even on exceptions.
- `EntityUtils.toByteArray()` — consume and close the entity stream properly.

### Monitoring (prevention)

After the incident, add monitoring to catch FD exhaustion before it becomes critical:

**Node-level alert (Prometheus rule):**

```yaml
- alert: NodeFileDescriptorExhaustion
  expr: node_filefd_allocated / node_filefd_maximum > 0.7
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Node file descriptor usage > 70%"
    description: "FD usage {{ $value | humanizePercentage }} on {{ $labels.instance }}"
```

**Per-process alert:**

```yaml
- alert: ProcessFileDescriptorHigh
  expr: process_open_fds / process_max_fds > 0.8
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Process FD usage > 80% on {{ $labels.instance }}"
```

**Grafana dashboard panel:**

Add a panel showing:
- `node_filefd_allocated` as a time series
- `node_filefd_maximum` as a reference line
- Top 5 processes by FD count (from `/proc` metrics)

**Manual check command (for ops runbook):**

```bash
# Quick FD health check
echo "=== System FD ===" && cat /proc/sys/fs/file-nr
echo "=== Top 5 processes by FD ==="
for pid in /proc/[0-9]*; do
  echo "$(basename $pid) $(ls "$pid"/fd 2>/dev/null | wc -l) $(cat "$pid"/comm 2>/dev/null)"
done | sort -k2 -rn | head -5
```

---

## Lessons Learned

### 1. Never trust defaults

The CentOS 7 default `ulimit -n` of **1024** was designed for an era when servers ran a handful of simple services. A modern Java microservice can easily exhaust this in seconds under load. Always explicitly set resource limits in systemd unit files.

### 2. Connection pools need limits

Every HTTP client, database connection pool, and network library must have:
- A **maximum size** (don't use defaults blindly)
- An **idle timeout** (evict stale connections)
- A **validated health check** (don't reuse dead connections)
- Proper **close handling** (try-with-resources or finally blocks)

### 3. Monitor, monitor, monitor

FD exhaustion is silent until it's catastrophic. Unlike CPU or memory, you don't get gradual degradation — you get a hard wall. Track `file-nr` and per-process FD counts in your monitoring system.

### 4. The cascade effect is real

One service's resource leak can take down an entire node. When `systemd-logind` can't open FDs, you can't SSH in to fix it. When a process can't `fork()`, you can't even run `kill`. This creates a **denial-of-service deadlock** that requires out-of-band access (IPMI, iDRAC, or a reboot) to resolve.

### 5. systemd vs /etc/security/limits.conf

Important distinction:
- **`/etc/security/limits.conf`** applies to PAM-based logins (SSH, console). It does NOT apply to systemd services.
- **systemd services** must use `LimitNOFILE=` in the unit file.
- Both are needed for comprehensive coverage.

---

## Summary

### Flow diagram

```
Deploy new order-service (with connection pool leak)
        │
        ▼
HTTP connections not closed → sockets accumulate in CLOSE_WAIT
        │
        ▼
Per-process FD count hits 1024 (default ulimit -n)
        │
        ▼
order-service stops serving → downstream services flood with retries
        │
        ▼
All 12 microservices on node exhaust FDs
        │
        ▼
systemd-logind fails → SSH unavailable
node is effectively dead
        │
        ▼
Kill order-service → kernel reclaims 64000+ FDs
        │
        ▼
All services recover immediately
```

### Timeline

| Time | Event |
|------|-------|
| 14:25 | Deploy `order-service` v2.3.1 with connection pool bug |
| 14:26 | FD count starts rising (leaking ~2 FDs/sec) |
| 14:32 | `order-service` hits 1024 FD limit, starts failing |
| 14:33 | Downstream services start cascading failures |
| 14:34 | Node-wide FD exhaustion begins (other services hit limits) |
| 14:35 | SSH becomes unreliable, alerts fire |
| 14:37 | On-call engineer kills the leaking process |
| 14:37 | All services recover, SSH restored |
| 15:00 | Limits increased, systemd unit updated |
| 18:00 | Code fix deployed and verified |

### Key commands cheat sheet

```bash
# System-wide FD stats
cat /proc/sys/fs/file-nr

# Per-process FD limits
cat /proc/<pid>/limits | grep "open files"

# Per-process FD count
ls -1 /proc/<pid>/fd | wc -l

# Top FD consumers on the system
for pid in /proc/[0-9]*; do
  echo "$(basename $pid) $(ls "$pid"/fd 2>/dev/null | wc -l)"
done | sort -k2 -rn | head -5

# Socket state summary
ss -tna | awk '{print $1}' | sort | uniq -c | sort -rn

# CLOSE_WAIT sockets for a process
lsof -p <pid> | grep CLOSE_WAIT

# Check systemd FD limits
systemctl show <service> | grep LimitNOFILE

# Set per-user limits (PAM)
echo "* soft nofile 65536" >> /etc/security/limits.conf
echo "* hard nofile 65536" >> /etc/security/limits.conf

# Set system-wide limit
echo "fs.file-max = 200000" >> /etc/sysctl.conf && sysctl -p
```

---

*File descriptor exhaustion is one of those problems that feels terrifying in the moment but is straightforward once you know where to look. The next time you see "Too many open files", don't panic — start with `file-nr`, find the offending PID, check if it's sockets in CLOSE_WAIT, and kill the leak. Then fix the code and set proper limits. You'll be back online in minutes.*
