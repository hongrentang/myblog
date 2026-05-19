---
title: "DNS Resolution Timeout Causing Microservice Latency Spikes"
date: 2026-05-19
weight: 100020
slug: "dns-resolution-timeout-troubleshooting"
tags: ["dns", "network", "coredns"]
categories: ["网络"]
description: "Full postmortem of a CoreDNS upstream timeout causing site-wide latency spikes — from misdiagnosing the application layer to pinpointing the DNS chain"
keywords: "dns resolution timeout, coredns timeout, kubernetes dns failure, dns troubleshooting, ndots configuration"
draft: false
featured: true
cover:
  image: "/images/dns-timeout-banner.svg"
  caption: "DNS Resolution Timeout — CoreDNS Upstream Failure"
---

# DNS Resolution Timeout Causing Microservice Latency Spikes

## Common Search Queries

If you're landing here from a search, this post covers:

- dns resolution timeout kubernetes
- coredns 5s timeout upstream
- microservice intermittent lookup failure
- ndots:5 causing excessive dns queries
- kubectl dns diagnostic commands

---

## The Incident

**Environment**: K8S v1.28, Calico CNI, CoreDNS 1.11, 200+ microservices.

**Time**: Wednesday 14:30, off-peak hours.

**Symptoms**: Monitoring alerted — order service P99 latency jumped from 50ms to 3.2s. A large number of requests returning 503. Upstream services reporting `connection refused`.

```bash
HTTP P99 Latency: 3240ms (threshold: 500ms)
Error Rate: 12.7% (threshold: 1%)
```

**Impact**: Users experienced 3-5 second wait times when placing orders. Some orders failed entirely.

---

## Misdiagnosis

### First Reaction: Application Layer Issue

Seeing P99 at 3.2s, my first instinct was upstream service overload or a slow database query. I immediately checked the order pod logs:

```bash
kubectl logs -n production order-service-7b8f9d4c5f-abc12 --tail 100
```

The logs showed extensive `connection refused` and `no such host` errors — but the target services were running fine, and their Endpoints were healthy. This meant: the services weren't down — they simply **couldn't be found**.

### Shifting Focus to DNS

```
2026-05-19T14:25:18.003Z ERROR Get "http://user-service.production.svc.cluster.local/api/check": dial tcp: lookup user-service.production.svc.cluster.local on 10.96.0.10:53: i/o timeout
```

Key detail: **i/o timeout**, target `10.96.0.10:53` — the CoreDNS Service IP.

---

## Investigation

### Step 1: Verify DNS Resolution Inside Pod

```bash
kubectl exec -n production order-service-7b8f9d4c5f-abc12 -- nslookup user-service.production.svc.cluster.local
```

Result:

```
Server:    10.96.0.10
Address:   10.96.0.10:53

** server can't find user-service.production.svc.cluster.local: NXDOMAIN
```

The full FQDN query returned NXDOMAIN, but sometimes it worked — classic **intermittent timeout** behavior.

### Step 2: Check Pod DNS Configuration

```bash
kubectl exec -n production order-service-7b8f9d4c5f-abc12 -- cat /etc/resolv.conf
```

```
nameserver 10.96.0.10
search production.svc.cluster.local svc.cluster.local cluster.local
ndots: 5
```

`ndots: 5` is the key. When a query name has fewer than 5 dots, Kubernetes appends each search domain sequentially. Querying `user-service` (0 dots) generates 4 DNS lookups:

1. `user-service.production.svc.cluster.local.`
2. `user-service.svc.cluster.local.`
3. `user-service.cluster.local.`
4. `user-service.` (absolute path)

Each goes through the full DNS resolver chain. If upstream times out, the delay multiplies.

### Step 3: Check CoreDNS Status

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7d6d9b7f4d-8k9m2  1/1     Running   0          12d
coredns-7d6d9b7f4d-x3p1q  1/1     Running   0          12d
```

Pods looked healthy, no restarts. On to the logs:

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail 100
```

```
[ERROR] plugin/errors: 2 user-service.production.svc.cluster.local. A: dial tcp 8.8.8.8:53: i/o timeout
[ERROR] plugin/errors: 2 user-service.svc.cluster.local. A: dial tcp 8.8.8.8:53: i/o timeout
[ERROR] plugin/errors: 2 user-service.cluster.local. A: dial tcp 8.8.8.8:53: i/o timeout
```

CoreDNS was configured to forward to `8.8.8.8` and `223.5.5.5`, but connectivity to `8.8.8.8` was intermittently failing. The default CoreDNS upstream timeout is 5s. With ndots multiplying queries: total stall = 5s × 4 attempts ≈ 20s worst case.

### Step 4: Check CoreDNS Config

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        forward . 8.8.8.8 223.5.5.5 {
          max_concurrent 1000
        }
        cache 30
        errors
    }
```

Two upstreams: `8.8.8.8` and `223.5.5.5`. The problem: `8.8.8.8` was being blocked by a firewall rule change on the internal network. The `forward` plugin tries upstreams sequentially, so a blocked upstream caused massive delays before falling back.

---

## Root Cause

1. CoreDNS had `8.8.8.8` as an upstream DNS, but **an internal firewall rule change** blocked TCP/53 to `8.8.8.8` from K8S nodes
2. The CoreDNS `forward` plugin waits for timeout (default 5s) before trying the next upstream
3. Pod `ndots: 5` caused 3-4 DNS queries per short name lookup, each waiting 5s
4. No application-level DNS caching — every request triggered fresh resolution

**Biggest misdiagnosis**: I spent the first 10 minutes analyzing application traces, convinced it was a slow database query causing cascading timeouts. It was DNS all along — every symptom (503s, latency spikes, connection refused) was a downstream effect of DNS failure.

---

## Recovery

### Hotfix (2 minutes)

Remove the unreachable upstream:

```bash
kubectl edit configmap coredns -n kube-system
```

Changed to:

```yaml
forward . 223.5.5.5 {
  max_concurrent 1000
}
```

Restart CoreDNS:

```bash
kubectl rollout restart -n kube-system deployment/coredns
```

Query latency dropped from 5s to 20ms. P99 latency recovered to 60ms within 1 minute.

---

## Long-Term Fixes

### 1. CoreDNS Configuration

Use only reachable upstreams with proper timeout and health checks:

```yaml
forward . 223.5.5.5 114.114.114.114 {
  max_concurrent 1000
  expire 10s
  policy sequential
  health_check 5s
}
```

### 2. Increase Cache TTL

Raise CoreDNS cache from 30s to 60s to reduce upstream query frequency:

```yaml
cache 60
```

### 3. Application-Level DNS Caching

Configure DNS caching in the application (JVM default behavior):

```bash
# JVM parameters
-Dsun.net.inetaddr.ttl=60
-Dsun.net.inetaddr.negative.ttl=10
```

### 4. Monitor Upstream Health

Deploy a DNS health checker DaemonSet:

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: dns-checker
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: dns-checker
  template:
    metadata:
      labels:
        app: dns-checker
    spec:
      containers:
      - name: checker
        image: busybox:1.36
        command:
        - sh
        - -c
        - "while true; do nslookup kubernetes.default.svc.cluster.local >/dev/null 2>&1; sleep 10; done"
EOF
```

---

## Reproduction Steps

1. Add an unreachable upstream to the CoreDNS ConfigMap
2. Restart CoreDNS
3. Enter any pod and run `nslookup some-service` — observe 5s+ latency
4. Check CoreDNS logs to confirm upstream timeout errors

---

## Quick Reference

```bash
# Test pod DNS resolution
kubectl exec -it <pod> -- nslookup <service.namespace.svc.cluster.local>

# View DNS configuration
kubectl exec -it <pod> -- cat /etc/resolv.conf

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail 50

# Test CoreDNS service directly
kubectl run -it --rm dns-test --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local 10.96.0.10

# View CoreDNS config
kubectl get configmap coredns -n kube-system -o yaml

# Restart CoreDNS
kubectl rollout restart -n kube-system deployment/coredns
```

---

## Summary

Investigation chain:

```
P99 spike → Misdiagnosed app slow query
  → Found connection refused + no such host
  → Tested DNS resolution (timeout)
  → Checked resolv.conf (ndots: 5)
  → Inspected CoreDNS logs (upstream 8.8.8.8 timeout)
  → Removed unreachable upstream → Recovery
```

Total time from alert to recovery: 12 minutes. First 10 minutes were spent on the wrong hypothesis.

**Lesson**: When latency spikes combine with `connection refused`, check DNS first — not the application layer. DNS is the cluster's lifeline. When it breaks, every service shows bizarre symptoms.
