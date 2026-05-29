---
title: "NetworkPolicy Failure — How Missing Pod Isolation Enabled Cluster-Wide Lateral Movement"
date: 2026-05-29
weight: 100220
slug: "networkpolicy-failure-lateral-movement"
tags: ["kubernetes", "security", "networkpolicy", "troubleshooting"]
categories: ["Security"]
description: "A Kubernetes network security incident — how a compromised frontend pod moved laterally to the database tier because no NetworkPolicy was in place, and why default-deny should be the default"
keywords: "kubernetes networkpolicy incident, k8s lateral movement, pod network isolation, default deny networkpolicy, zero trust kubernetes network"
draft: false
featured: true
cover:
  image: ""
  caption: "NetworkPolicy Failure — Lateral Movement Incident"
---

# NetworkPolicy Failure — How Missing Pod Isolation Enabled Cluster-Wide Lateral Movement

## Common Search Queries

- kubernetes networkpolicy lateral movement
- k8s pod to pod network isolation
- default deny networkpolicy kubernetes
- zero trust networking kubernetes
- how to prevent lateral movement in k8s

---

## The Incident

**Environment**: K8S v1.28, Calico CNI, 200+ microservices across 15 namespaces, 3-year-old cluster.

**Time**: 02:10 AM. IDS alerted on unusual database traffic patterns from an unexpected source.

**Initial Symptoms**: The production PostgreSQL database was receiving queries from a frontend Node.js pod — something that had never happened before. The database was only supposed to be accessed by backend API services.

```bash
# Suspicious connection detected by the IDS
Source: frontend/web-app-7d9f8c6b-x (10.244.3.15)
Target: database/postgres-primary-0 (10.244.1.5)
Port: 5432 (PostgreSQL)
Protocol: TCP
Query: SELECT * FROM users; DROP TABLE payments;  # ← Malicious
```

**Impact**: Production database compromised. Customer payment records partially deleted. 6 hours of data restoration from backup.

---

## Background

When this cluster was first deployed three years ago, the team was in a hurry to ship. Kubernetes NetworkPolicy was on the roadmap — but it was deprioritized repeatedly. "We'll get to it after the next sprint."

The result: zero NetworkPolicies across the entire cluster. Every pod could talk to every other pod:

```bash
# With no NetworkPolicy, this works from ANY pod:
kubectl exec -it frontend/web-app-abc -- bash
curl database-svc.production.svc.cluster.local:5432  # ← No policy blocks this
curl redis-svc.cache.svc.cluster.local:6379            # ← No policy blocks this
```

A frontend pod compromised through an RCE vulnerability in the web framework could reach any service in the cluster.

The attacker exploited CVE-2026-12345 (a known RCE in the Node.js Express framework) in the frontend application. From there, they scanned the internal network:

```bash
# From inside the compromised frontend pod
for svc in api-gateway database redis payment auth; do
  for ns in production staging cache; do
    host="${svc}.${ns}.svc.cluster.local"
    if timeout 1 bash -c "echo >/dev/tcp/$host/80" 2>/dev/null; then
      echo "OPEN: $host:80"
    fi
  done
done
```

They found 40+ internal services reachable from the frontend. The flat network gave them carte blanche.

---

## Investigation

### Step 1: Identify the Entry Point

```bash
# Check which pod initiated the malicious database query
grep "10.244.3.15" /var/log/kubernetes/audit.log | grep "5432" | head -5
```

Traced back to `frontend/web-app-7d9f8c6b-x`. The pod had an RCE exploit from an unpatched library.

### Step 2: Map the Blast Radius

```bash
# From the incident response pod, check what internal services were reachable from that node
kubectl get pods -n production -o wide | grep 10.244.3
# Check for unusual connections in the last 24 hours
```

The attacker had connected to:
- `database` (PostgreSQL) — extracted and deleted payment records
- `redis` (cache) — dumped session tokens
- `payment-api` (payment processing) — attempted refund transactions
- `auth` (authentication) — attempted credential extraction

### Step 3: Verify No NetworkPolicies Existed

```bash
kubectl get networkpolicy --all-namespaces
# No resources found — entire cluster had zero network policies
```

```bash
# Confirm: any pod could reach any service
kubectl run test-pod --image=busybox -it --rm -- wget -qO- http://database-svc.production:5432
# Connection succeeded — full access
```

---

## Root Cause

1. **No NetworkPolicies defined**: The cluster had been running for 3 years without a single NetworkPolicy resource
2. **Flat network by default**: Kubernetes allows all pod-to-pod traffic by default. Without explicit policies, every pod can reach every other pod
3. **No defense-in-depth**: Even if the RCE vulnerability was eventually patched, NetworkPolicies would have contained the blast radius to the frontend tier only
4. **No east-west traffic monitoring**: The IDS only monitored north-south traffic (ingress/egress). Internal lateral movement was invisible until the database was already compromised

---

## Resolution

### Emergency (Containment)

```bash
# Immediately isolate the compromised namespace
kubectl label namespace production pod-security.kubernetes.io/enforce=restricted

# Delete and recreate the compromised frontend deployment with fix
kubectl delete deployment web-app -n frontend
# Re-deploy with patched image after vulnerability fix
kubectl apply -f frontend-deployment-patched.yaml

# Force rotate all database credentials
kubectl exec -it postgres-primary-0 -n production -- psql -c "ALTER USER app_user WITH PASSWORD 'new-password';"

# Invalidate all Redis sessions
kubectl exec -it redis-master-0 -n cache -- redis-cli FLUSHALL

# Restore database from backup before the deletion
kubectl exec -it postgres-primary-0 -n production -- psql -f /backups/2026-05-28-full.sql
```

### Deploy Default-Deny NetworkPolicies

```yaml
# Default deny all ingress traffic (applied to all namespaces)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Default deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

### Implement Tier-Based Microsegmentation

```yaml
# Frontend tier: only accepts internet traffic, only talks to API tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-network
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 0.0.0.0/0
      ports:
      - port: 443
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: api
    ports:
    - port: 8080
    - port: 8443
---
# API tier: only accepts traffic from frontend, talks to database and cache
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network
  namespace: api
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  - to:
    - namespaceSelector:
        matchLabels:
          tier: cache
    ports:
    - port: 6379
---
# Database tier: only accepts traffic from API tier
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-network
  namespace: database
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: api
    ports:
    - port: 5432
```

### Label Namespaces for Policy Selectors

```bash
kubectl label namespace frontend tier=frontend
kubectl label namespace api tier=api
kubectl label namespace database tier=database
kubectl label namespace cache tier=cache
```

### Verification

```bash
# Verify: frontend can NOT reach database directly
kubectl run test-pod --image=busybox -n frontend -it --rm -- wget -qO- http://database-svc.database:5432
# Connection timed out — policy is working

# Verify: api can reach database
kubectl run test-pod --image=busybox -n api -it --rm -- wget -qO- http://database-svc.database:5432
# Connection succeeded — correct access
```

### Long-Term Prevention

1. **NetworkPolicy as part of CI/CD**: Reject deployments that don't include NetworkPolicy manifests
2. **Regular network audit**: Use `kubectl netpol` or `kubeaudit` to scan for namespaces without policies
3. **Zero-trust networking**: Default-deny for all new namespaces via admission controller
4. **East-west traffic monitoring**: Deploy service mesh (Istio/Linkerd) or eBPF-based monitoring (Cilium) for internal traffic visibility
5. **Vulnerability scanning**: Patch known CVEs faster — the RCE used here had a fix available for 3 weeks

---

## Lessons Learned

- **Kubernetes flat network is NOT secure by default**: "Pods can talk to everything" is the default. Security requires explicit NetworkPolicy configuration
- **NetworkPolicy is your blast radius containment**: Even if an application vulnerability exists, NetworkPolicy prevents the attacker from reaching other tiers
- **Default-deny first, then allow specific**: Start with `default-deny-ingress` and `default-deny-egress`, then add fine-grained allow rules
- **Defense in depth**: Don't rely on a single security control. NetworkPolicy + runtime security + vulnerability scanning + monitoring work together
- **Label namespaces for policy selectors**: Use a consistent tier-based labeling scheme (`tier: frontend`, `tier: api`, `tier: database`) to write clean, maintainable policies

---

## Summary

The attack chain:

```
Frontend pod with unpatched RCE vulnerability (CVE-2026-12345)
→ Attacker gains shell access to frontend container
→ Scans internal network: finds 40+ reachable services
→ Connects to PostgreSQL (no NetworkPolicy blocked it)
→ Extracts and deletes payment records
→ Dumps Redis session tokens
→ Attempts fraudulent transactions via payment API
```

Containment: 30 minutes. Data restoration: 6 hours. Fix (NetworkPolicy): 2 days deployment. The RCE was the spark — but the flat network was the fuel. NetworkPolicy isolates. Default-deny first.
