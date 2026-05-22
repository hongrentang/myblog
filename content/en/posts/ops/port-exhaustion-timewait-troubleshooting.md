---
title: "TCP Port Exhaustion — From 'Cannot assign requested address' to TIME_WAIT Disaster"
date: 2026-05-22
weight: 100130
slug: "port-exhaustion-timewait-troubleshooting"
tags: ["tcp", "network", "port-exhaustion", "timewait"]
categories: ["网络"]
description: "A real-world debugging journey from 78% payment callback failure rate, through DNS and backend blind alleys, to finding TIME_WAIT buildup exhausting all ephemeral ports"
keywords: "tcp port exhaustion, timewait, ephemeral port, connection pool, cannot assign requested address"
draft: false
featured: true
cover:
  image: "/images/port-exhaustion-banner.svg"
  caption: "TCP Port Exhaustion Troubleshooting"
---

# TCP Port Exhaustion — From "Cannot assign requested address" to TIME_WAIT Disaster

## Been There?

01:47 AM. Alarm group explodes.

"Payment callback push failure rate: 78%." That number got me out of bed faster than coffee.

Quick log check:
- Service A (Python/Gunicorn) receives payment callbacks and forwards them to internal settlement Service B
- Logs are flooding: `OSError: [Errno 99] Cannot assign requested address`
- Not intermittent — it's staying broken

First thought: is Service B down? Network issue?

## Investigation

### Wrong turn 1: Blaming the target

SSH to the box, curl the target directly:

```bash
curl -v http://10.0.0.50:8080/health
```

```
*   Trying 10.0.0.50:8080...
* Connected to 10.0.0.50 (10.0.0.50) port 8080 (#0)
> GET /health HTTP/1.1
> Host: 10.0.0.50:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not-in-connection 0
< HTTP/1.1 200 OK
```

Health check responds fine. Service B is alive. Multiple curls confirm it.

So it's not the target.

### Wrong turn 2: Suspecting DNS

Service A calls Service B by domain name. Maybe DNS is serving bad records?

```bash
dig +short payment-internal.svc.cluster.local
```

```
10.0.0.50
```

IP is correct.

```bash
dig +short payment-internal
```

```
10.0.0.50
```

DNS is fine.

**Lesson learned**: This cost me about 10 minutes. When "connection fails," the brain defaults to checking the remote side — is it down, can I resolve it? But that instinct leads you astray when the problem isn't "can't reach them" but "**you have no resources left to reach anyone**." It's like having a phone that says "no SIM card available" — their number is right, the signal is fine, you just can't make the call.

### Wrong turn 3: Checking connection pools / thread pools

Reading the logs more carefully: the error isn't "connection refused" (target rejected) — it's "Cannot assign requested address." That smells like port exhaustion.

Time for `ss`:

```bash
ss -s
```

```
Total: 40231 (kernel)
TCP:   22340 (estab 89, closed 22192, timewait 22101, ...)
```

22101 TIME_WAIT connections?! That's not a typo. Twenty-two thousand connections stuck in TIME_WAIT on a single machine.

```bash
# Where are they going?
ss -tan | awk '{print $4,$5}' | sort | uniq -c | sort -rn | head -10
```

```
21084 10.0.0.10:45231 10.0.0.50:8080
  312 10.0.0.10:45232 10.0.0.50:8080
  298 10.0.0.10:45233 10.0.0.50:8080
  ...
```

Every single one is from the local host (10.0.0.10) to Service B (10.0.0.50:8080). And they all use different source ports — that's the smoking gun. Each request opens a new connection.

### The breakthrough: ephemeral port exhaustion

When Linux initiates a TCP connection, the kernel assigns a temporary (ephemeral) source port. Default range:

```bash
cat /proc/sys/net/ipv4/ip_local_port_range
```

```
32768 60999
```

Available ports = 60999 - 32768 = **28,231**.

We have 22,101 in TIME_WAIT, plus active ESTABLISHED connections. That leaves ~6,000 free. At the current request rate, those would be consumed in seconds.

```bash
# Confirm available port count
echo $((60999 - 32768 - $(ss -tan state time-wait | wc -l)))
```

It was already negative.

Let's confirm what the process is actually doing:

```bash
strace -e trace=network -p $(pgrep -f gunicorn) 2>&1 | head -20
```

```
connect(33, {sa_family=AF_INET, sin_port=htons(8080), sin_addr=inet_addr("10.0.0.50")}, 16) = -1 EADDRNOTAVAIL (Cannot assign requested address)
```

Confirmed: `connect()` returns `EADDRNOTAVAIL` — the kernel has no free ephemeral ports.

### Why did ports run dry?

The code. Service A used the `requests` library directly:

```python
import requests

def forward_payment(data):
    resp = requests.post("http://payment-internal:8080/orders", json=data)
    return resp.json()
```

Every `forward_payment()` call opens a brand new TCP connection. When the request finishes, the connection closes and enters TIME_WAIT.

At peak, payment callbacks hit hundreds of requests per second. Each generates a new connection. TIME_WAIT lasts 60 seconds. 60s × 100 qps = **6,000 connections** stuck in TIME_WAIT at any moment.

With 28,231 ports available, it takes under a minute to exhaust them. After that, every new connection fails.

```bash
# Check port range distribution
ss -tan state time-wait | awk '{print $4}' | cut -d: -f2 | sort -n | head -5
```

```
32768
32769
32800
32850
...
```

Ports are allocated sequentially from 32768 and have already reached 60000+ — the pool has been swept through.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct | Every HTTP request creates a new TCP connection; high concurrency drives TIME_WAIT buildup and exhausts ephemeral ports |
| Code flaw | Using raw `requests` calls without Session/connection pooling |
| Trigger | Payment callback traffic spike (hundreds of concurrent requests) |
| Why isolated | Only the Service A → Service B path used this pattern; other internal calls already used connection pooling |

**One sentence**: The code creates a new connection per request, TIME_WAIT piles up, and the kernel's ephemeral port pool runs dry.

## Fix

### Emergency stopgap: expand port range

Get the alarm to shut up first:

```bash
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
```

Persist:

```
net.ipv4.ip_local_port_range = 1024 65535
```

Available ports go from 28,231 to 64,511 — buys us time.

```bash
sysctl -p
```

Enable TIME_WAIT reuse (works for client-side outgoing connections):

```bash
sysctl -w net.ipv4.tcp_tw_reuse=1
```

`tcp_tw_reuse` lets the kernel allocate a TIME_WAIT port for a new outbound connection if TCP timestamps indicate it's safe. Since our problem is **outgoing** (Service A → Service B), this is safe and effective.

### Root fix: connection pooling (code change)

Expanding ports is a band-aid. The real fix is to stop creating so many connections. Use `requests.Session`:

```python
import requests
from urllib3.util.retry import Retry
from requests.adapters import HTTPAdapter

# Session with connection pool
session = requests.Session()

adapter = HTTPAdapter(
    pool_connections=50,
    pool_maxsize=200,
    pool_block=True,
    max_retries=Retry(
        total=2,
        backoff_factor=0.5,
        status_forcelist=[502, 503, 504]
    )
)
session.mount("http://", adapter)
session.mount("https://", adapter)

def forward_payment(data):
    resp = session.post(
        "http://payment-internal:8080/orders",
        json=data,
        timeout=5
    )
    return resp.json()
```

**Why connection pooling works**:

| Mechanism | Effect |
|-----------|--------|
| Connection reuse | Multiple requests share the same TCP connection; no more 3-way handshake per request |
| Eliminates TIME_WAIT | Connections stay open, so they never enter TIME_WAIT |
| Lower latency | Skips TCP handshake RTT on every call |
| No port exhaustion | 50 long-lived connections handle hundreds of QPS easily |

After deploying:

```bash
# Check TIME_WAIT count 5 minutes post-deploy
ss -tan state time-wait | wc -l
```

```
234
```

From 22,101 to 234.

```bash
ss -tan | grep 8080
```

```
ESTAB  10.0.0.10:45231  10.0.0.50:8080
ESTAB  10.0.0.10:45232  10.0.0.50:8080
ESTAB  10.0.0.10:45233  10.0.0.50:8080
ESTAB  10.0.0.10:45234  10.0.0.50:8080
...
```

All ESTAB, zero TIME_WAIT. Connections are being reused.

### Verification

```bash
# 1. Error log check
grep "Cannot assign requested address" /var/log/app/error.log
# Empty = gone

# 2. TIME_WAIT count
ss -tan state time-wait | wc -l
# Should be well under 28231

# 3. Payment success rate
# From 78% failure to <0.1% (residual network jitter)

# 4. Load test
ab -n 1000 -c 100 http://localhost:8080/call-b
# Zero errors = fix confirmed
```

✅ **Recovery confirmed**:
- No more `EADDRNOTAVAIL` in logs
- TIME_WAIT dropped from 22,101 to 234
- Payment callback success rate back to 99.9%+

### Long-term prevention

```bash
# Monitor TCP connection states
# Prometheus: node_netstat_Tcp_TimeWait (from node_exporter)
# Alert when TIME_WAIT > 10000

# Code review rule
# Every HTTP call MUST use Session/connection pooling
# Consider gRPC for inter-service calls (built-in connection reuse)

# Harden kernel params
cat >> /etc/sysctl.d/99-network-tuning.conf << 'EOF'
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.core.somaxconn = 4096
EOF

sysctl --system
```

## Hindsight: what I'd do differently

**First command should have been `ss -s`.**

Not DNS lookup, not curl to backend, not even the log file. Just check system-level connection state. "Total: 40231, TIME_WAIT: 22101" would have pointed direction in 3 seconds.

Network failures fall into two categories: (1) the target is broken, or (2) **you ran out of local resources**. Port exhaustion, connection pool starvation, fd leaks — these all belong to category 2. They only surface under load, and they laugh at low-traffic curl tests.

## What I Learned

1. **"Cannot assign requested address" is not a permissions error.** First time I saw this, I went down a rabbit hole checking selinux and capabilities. `EADDRNOTAVAIL` just means "no kernel ports left" — you've been consumed by your own success.

2. **Raw `requests` calls in production are an anti-pattern.** Every `requests.get()` is a new TCP connection. Use `requests.Session`. This isn't performance tuning — it's correctness. Without connection pooling, high concurrency will exhaust your ports eventually. It's a question of when, not if.

3. **TIME_WAIT isn't a bug, but accumulating 20,000 of them is.** TCP's TIME_WAIT exists for a good reason (ensuring the peer receives the final ACK). A handful is normal. Thousands targeting the same endpoint? Your code isn't reusing connections.

4. **Load tests must stress the connection layer.** Functional tests won't catch port exhaustion — QPS isn't high enough. Pre-deployment load testing should at minimum check: TIME_WAIT count, local port utilization, and connection establishment rate. Any of these three going abnormal means an incident waiting to happen.
