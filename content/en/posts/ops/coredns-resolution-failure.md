---
title: "CoreDNS Resolution Failure — When Kubernetes Services Became Unreachable by Name"
date: 2026-06-07
weight: 100390
slug: "coredns-resolution-failure"
tags: ["coredns", "kubernetes", "troubleshooting", "dns", "network"]
categories: ["Troubleshooting"]
description: "A CoreDNS resolution failure incident — how a misconfigured ndots value and CoreDNS pod resource exhaustion caused widespread DNS resolution failures across the Kubernetes cluster"
keywords: "coredns troubleshooting, kubernetes dns resolution failure, coredns resource limits, ndots kubernetes, dns timeout kubernetes pod"
draft: false
featured: true
cover:
  image: ""
  caption: "CoreDNS Resolution Failure — Troubleshooting"
---

# CoreDNS Resolution Failure — When Kubernetes Services Became Unreachable by Name

## Common Search Queries

If you're landing here from a search, this post covers:

- coredns resolution failure kubernetes
- kubernetes dns intermittent failure some pods work others don't
- coredns resource limits not set throttled
- ndots:5 causing excessive dns queries search domain expansion
- coredns auto-scaling replicas insufficient for large cluster
- kubectl check coredns dns resolution troubleshooting

---

## The Incident

**Environment**: K8S v1.28, Calico CNI, CoreDNS 1.10.1, 50 worker nodes, 500+ services, 2000+ pods.

**Time**: Tuesday 10:30, morning business peak.

**Symptoms**: The monitoring dashboard lit up — suddenly dozens of services began reporting DNS resolution failures. Applications failing with `Name or service not known` errors when connecting to peer services. Some pods within the same Deployment could resolve DNS names successfully while their counterparts could not. The failures were intermittent and grew worse as load increased.

```bash
# Alerts triggered in the first 5 minutes
Service: order-svc  |  Error: dial tcp: lookup payment-svc on 10.96.0.10:53: no such host
Service: api-gateway |  Error: dial tcp: lookup user-svc on 10.96.0.10:53: i/o timeout
Service: notification | Error: dial tcp: lookup alertmanager on 10.96.0.10:53: no such host
```

**Impact**: The checkout and payment flows were completely broken for approximately 12 minutes during the morning peak. Approximately 15% of all internal service-to-service calls failed, causing cascading partial outages across three critical business domains. Revenue-affecting incident declared at 10:33.

---

## Background

### CoreDNS Architecture in Kubernetes

CoreDNS is the default DNS resolver in Kubernetes clusters since v1.13. It runs as a Deployment in the `kube-system` namespace and is exposed via the `kube-dns` ClusterIP service (typically `10.96.0.10`). Every pod in the cluster is configured to use this service IP as its nameserver via the `/etc/resolv.conf` injected by kubelet.

```
Pod → /etc/resolv.conf (nameserver 10.96.0.10) → kube-dns Service → CoreDNS Pod → Upstream DNS
```

CoreDNS serves both:
- **Cluster DNS**: resolving Kubernetes Service names (`service.namespace.svc.cluster.local`)
- **External DNS**: forwarding non-cluster queries to upstream resolvers (e.g., `/etc/resolv.conf` on the node)

### DNS Resolution in Pods — The /etc/resolv.conf

When a pod starts, kubelet generates its `/etc/resolv.conf` with a specific structure:

```
nameserver 10.96.0.10
search <namespace>.svc.cluster.local svc.cluster.local cluster.local <node-search-domains>
options ndots:5
```

This `search` line and the `ndots` option are the key players in this story. They control how DNS name resolution works for **short names** (names that are not fully qualified domain names, i.e., not ending with a dot).

### What Is ndots?

The `ndots` option tells the DNS resolver: "if the name being resolved has fewer than N dots, try appending each search domain before querying it as-is." With `ndots:5` (the Kubernetes default):

- A short name like `payment-svc` (0 dots) triggers **multiple DNS queries**:
  1. `payment-svc.<namespace>.svc.cluster.local.`
  2. `payment-svc.svc.cluster.local.`
  3. `payment-svc.cluster.local.`
  4. `payment-svc.<node-search-domain-1>.`
  5. `payment-svc.<node-search-domain-2>.`
  6. `payment-svc.` (absolute query — only if all above fail)

That is **6 DNS queries** for a single short-name lookup. Now multiply this by hundreds of services and thousands of requests per second.

---

## Investigation

### Step 1: Basic DNS Test from a Debug Pod

The first thing any Kubernetes engineer reaches for — run a debug pod and test basic resolution:

```bash
kubectl run test-dns --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes.default.svc.cluster.local
```

Result — **intermittent**:

```
# First attempt — success
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes.default.svc.cluster.local
Address 1: 10.96.0.1 kubernetes.default.svc.cluster.local

# Second attempt 30 seconds later — failure
;; connection timed out; no servers could be reached
```

This confirmed DNS was the problem. But why intermittent?

### Step 2: Check CoreDNS Pod Status

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                       READY   STATUS    RESTARTS   AGE
coredns-7d6f9b7c8f-ab12c   1/1     Running   0          12d
coredns-7d6f9b7c8f-xy34d   1/1     Running   3          12d
```

Two replicas. One had been restarted 3 times. That was suspicious.

### Step 3: Inspect CoreDNS Logs

```bash
kubectl logs -n kube-system coredns-7d6f9b7c8f-ab12c --tail 100
```

```
[INFO] 10.244.1.15:33112 - 60130 "A IN payment-svc.production.svc.cluster.local. udp 72 false 512" NXDOMAIN qr,rd,ra 138 0
[INFO] 10.244.1.15:33112 - 60131 "A IN payment-svc.production.svc.cluster. udp 60 false 512" NXDOMAIN qr,rd,ra 126 0
[INFO] 10.244.1.15:33112 - 60132 "A IN payment-svc.production.svc. udp 48 false 512" NXDOMAIN qr,rd,ra 114 0
[INFO] 10.244.1.15:33112 - 60133 "A IN payment-svc.production. udp 38 false 512" NXDOMAIN qr,rd,ra 104 0
[WARNING] plugin/forward: connect to upstream: connection refused
[ERROR] plugin/errors: 2 5992878037655751999.5355688120039757079. Hinfo: unreachable backend: read udp 10.244.1.10:42135->10.0.0.1:53: i/o timeout
```

We could see:
- The search domain expansion in action — consecutive A-record queries with progressively shorter search suffixes
- Upstream forward failures — `connection refused` and `i/o timeout`

### Step 4: Check Resource Usage

```bash
kubectl top pods -n kube-system -l k8s-app=kube-dns
```

```
NAME                       CPU(cores)   MEMORY(bytes)
coredns-7d6f9b7c8f-ab12c   385m         287Mi
coredns-7d6f9b7c8f-xy34d   412m         312Mi
```

Nearly 400m CPU each — significant for a DNS pod. Let's check resource limits:

```bash
kubectl get pod -n kube-system coredns-7d6f9b7c8f-ab12c -o yaml | grep -A 6 resources
```

```
resources:
  requests:
    cpu: 100m
    memory: 70Mi
```

No limits were set — only requests. This meant CoreDNS pods could be **throttled by the kernel (CFS)** when node CPU was under pressure, and could be **evicted** under memory pressure.

### Step 5: Check /etc/resolv.conf Inside a Failing Pod

```bash
kubectl exec -n production payment-v1-7b8f9d4c5f-abc12 -- cat /etc/resolv.conf
```

```
nameserver 10.96.0.10
search payment.svc.cluster.local svc.cluster.local cluster.local prod-ns-1.svc.corp.local prod-ns-2.svc.corp.local
options ndots:5
```

Two problems visible:
1. **ndots:5** — causing up to 6 search domain lookups per query
2. **Long search line** — inherited from the node's own `/etc/resolv.conf` (the node had custom search domains from the corporate network). This meant even more search domain iterations.

### Step 6: Test with Explicit FQDN vs Short Name

Inside the same pod:

```bash
# Short name — 6+ queries, slow, sometimes fails
nslookup payment-svc
# Server: 10.96.0.10
# ** server can't find payment-svc: NXDOMAIN

# FQDN (ends with dot) — single query, works
nslookup payment-svc.production.svc.cluster.local.
# Server: 10.96.0.10
# Address 1: 10.96.0.1
# Name: payment-svc.production.svc.cluster.local
# Address 1: 10.244.3.12 payment-svc.production.svc.cluster.local
```

This proved the search domain expansion was the bottleneck.

### Step 7: Check CoreDNS ConfigMap

```bash
kubectl get configmap -n kube-system coredns -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus
        forward . /etc/resolv.conf
        cache 30
    }
```

Notable observations:
- No `max_concurrent` set on the forward plugin — no limit on concurrent upstream connections
- Cache set to only 30 seconds — very short TTL for a cluster with 500+ services
- No autoscaling or pod-level resource limits

---

## Root Cause

Three interconnected factors conspired to bring DNS down:

### Factor 1: CoreDNS Pods Without Resource Limits

The CoreDNS pods had **resource requests but no limits**. This is a dangerous configuration in production:

- **No CPU limits**: Under node CPU pressure (which is common on 50-node clusters during peak hours), CoreDNS pods experienced severe CFS throttling. A DNS query that should take <1ms could take 100ms+ or time out entirely.
- **No memory limits**: When node memory ran low, CoreDNS pods were prime candidates for eviction. The 3 restarts on one replica were OOMKills.

```bash
# Check previous pod termination reason
kubectl get pod -n kube-system coredns-7d6f9b7c8f-xy34d -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# OOMKilled
```

### Factor 2: Default ndots:5 Causing 6x Query Amplification

With `ndots:5` and the default search domains (plus extra domains inherited from the node's resolv.conf), each short-name DNS lookup generated **6 or more upstream queries** before succeeding or failing. For a cluster processing thousands of internal requests per second:

```
Normal:  1 query per lookup  →  1000 lookups/s  →  1000 qps on CoreDNS
ndots:5: 6 queries per lookup → 1000 lookups/s →  6000 qps on CoreDNS
```

This 6x amplification overwhelmed the already-throttled CoreDNS pods. The `forward` plugin couldn't keep up, causing connection timeouts to upstream resolvers.

### Factor 3: Insufficient CoreDNS Replicas

Two CoreDNS replicas for a 50-node cluster with 500+ services is grossly inadequate. The ClusterProportionalAutoscaler was not configured. Each replica was handling ~3000 qps during peak hours — far beyond the recommended 1000-2000 qps per CoreDNS instance.

---

## Resolution

### Emergency Response (Immediate Mitigation)

1. **Scale up CoreDNS replicas** to immediately relieve pressure:

```bash
kubectl scale deployment -n kube-system coredns --replicas=5
```

Within 2 minutes, DNS resolution stabilized. Error rates dropped from 15% to <0.1%.

2. **Set resource limits** on CoreDNS pods to prevent throttling and ensure QoS class:

```bash
kubectl edit deployment -n kube-system coredns
```

```yaml
resources:
  requests:
    cpu: 100m
    memory: 70Mi
  limits:
    cpu: 500m
    memory: 500Mi
```

### Fix ndots Configuration

Three approaches, any of which solves the problem:

**Option A — Set ndots:1 in the cluster DNS ConfigMap** (global fix):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubelet-config
  namespace: kube-system
data:
  kubelet: |
    resolvConf: /etc/resolv.conf
    ...
```

Or more practically — configure the pod `dnsConfig` in your workloads:

```yaml
spec:
  template:
    spec:
      dnsConfig:
        options:
          - name: ndots
            value: "1"
```

With `ndots:1`, a name like `payment-svc` (0 dots, which is < 1) still triggers search domain expansion. So you actually need `ndots:0` — or better yet, use FQDNs.

**Option B — Use FQDN in application configuration** (recommended):

Change your application configs to use fully qualified domain names with a trailing dot:

```
http://payment-svc.production.svc.cluster.local./api/v1/charge
```

This bypasses search domain expansion entirely — a single DNS query, no ndots processing.

**Option C — Patch the pod spec DNS policy**:

```yaml
spec:
  template:
    spec:
      dnsPolicy: "Default"  # Inherit node's DNS, ignoring kubelet search domains
      dnsConfig:
        options:
          - name: ndots
            value: "1"
```

### Update CoreDNS ConfigMap for Better Performance

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }
        prometheus
        forward . /etc/resolv.conf {
            max_concurrent 1000
        }
        cache 120
    }
```

Key changes:
- **`max_concurrent 1000`** — limits concurrent upstream connections to prevent resource exhaustion
- **`cache 120`** — increased from 30 to 120 seconds to improve cache hit ratios

### Add ClusterProportionalAutoscaler

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/cluster-proportional-autoscaler/master/manifests/coredns.yaml
```

This automatically scales CoreDNS replicas based on cluster node count:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: ClusterProportionalAutoscaler
metadata:
  name: coredns-autoscaler
  namespace: kube-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: coredns
  options:
    target: "NodeCount"
    base: 2
    max: 8
    nodesPerReplica: 10
```

For 50 nodes: `base(2) + ceil(50/10) = 7` replicas — sufficient for the load.

### Monitoring and Alerting

Add these CoreDNS metrics to your monitoring stack:

| Metric | What It Tells You | Alert Threshold |
|---|---|---|
| `prometheus_coredns_dns_responses_total` | Total DNS responses by rcode | NXDOMAIN rate > 5% |
| `prometheus_coredns_dns_request_duration_seconds` | Query latency histogram | p99 > 1s |
| `prometheus_coredns_forward_requests_total` | Upstream forward requests | Dropped > 1% |
| `prometheus_coredns_cache_hits_total` | Cache efficiency | Hit ratio < 50% |
| `prometheus_coredns_panics_total` | Plugin panics | > 0 |

---

## Lessons Learned

### What Went Wrong

1. **No resource limits on CoreDNS**: A control-plane component running without limits is a recipe for disaster under load. CoreDNS is especially sensitive because DNS is synchronous and connection-oriented (UDP streams and TCP fallback).

2. **Default ndots:5 is wasteful**: While the Kubernetes default of `ndots:5` works for small clusters, it creates massive query amplification in large environments. The default is designed to make short names "just work", but the cost becomes prohibitive at scale.

3. **CoreDNS autoscaling is not optional**: For any cluster with more than 10 nodes, manual replica management of CoreDNS is insufficient. Load varies with service count, request rate, and node count — all dynamic factors that demand autoscaling.

4. **Inheriting node resolv.conf search domains**: When nodes have custom DNS search domains (common in corporate environments), these leak into pods and expand the search list, multiplying the ndots problem.

### What We Improved

- CoreDNS now has **both requests and limits** configured, with Guaranteed QoS class
- All cluster workloads adopted **ndots:1** or FQDN-based addressing
- ClusterProportionalAutoscaler manages CoreDNS replicas dynamically
- DNS monitoring dashboard with latency, error rate, and cache hit ratio panels
- Standardized pod `dnsConfig` across all team Helm charts to explicitly set ndots

---

## Summary

This incident was a perfect storm of three manageable issues compounding into a production outage:

| Issue | Symptom | Fix |
|---|---|---|
| No resource limits on CoreDNS | Pods throttled/evicted under node pressure | Set CPU/memory limits, Guaranteed QoS |
| Default ndots:5 | 6x query amplification per short-name lookup | Set ndots:1 or use FQDNs |
| Only 2 CoreDNS replicas for 50 nodes | Pods overwhelmed at peak qps | ClusterProportionalAutoscaler → 7 replicas |

The most deceptive aspect of this failure was its **intermittent nature** — the same service would fail to resolve `payment-svc` one second and succeed the next, depending on which CoreDNS replica handled the query and whether that replica was currently being throttled. This type of "heisenbug" makes DNS issues particularly frustrating to diagnose.

After all fixes were applied, the cluster's DNS p99 latency dropped from 2.3s to 8ms, and cache hit ratios improved from ~35% to ~85%. The CoreDNS pods now handle peak traffic comfortably at ~30% CPU utilization with 5-7 replicas.

**Key takeaway**: Kubernetes DNS is not "free infrastructure" — it is a critical-path distributed system that demands proper sizing, resource guarantees, and explicit configuration. The default settings (ndots:5, no limits, no autoscaling) are acceptable for dev clusters but will fail under production load. When debugging intermittent DNS failures, always check `/etc/resolv.conf` inside the pod, CoreDNS resource usage, and the search domain expansion behavior.
