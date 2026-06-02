---
title: "API Server Unauthorized Access — How a Publicly Exposed kube-nginx Almost Cost the Cluster"
date: 2026-06-03
weight: 100280
slug: "api-server-unauthorized-access"
tags: ["kubernetes", "security", "api-server", "troubleshooting"]
categories: ["Security"]
description: "A Kubernetes API Server security incident — how an exposed API server without proper authentication allowed anonymous API calls, and why OIDC + RBAC + audit logging are the minimum defense"
keywords: "kubernetes api server exposed, anonymous auth kubernetes, kube-apiserver security, oidc kubernetes auth, kubectl unauthorized access"
draft: false
featured: true
cover:
  image: ""
  caption: "API Server Unauthorized Access — Incident Response"
---

# API Server Unauthorized Access — How a Publicly Exposed kube-nginx Almost Cost the Cluster

## Common Search Queries

- kubernetes api server exposed public internet
- anonymous auth kubernetes api
- kube-apiserver authentication bypass
- secure kubernetes api server
- kubectl without authentication

---

## The Incident

**Environment**: K8S v1.27, kubeadm deployed, single control plane node (2C/4G), used as a development cluster. API Server behind a kube-nginx reverse proxy.

**Time**: Friday 22:00. Cloudflare Security Center alerted: `https://k8s-api.dev.example.com` was being accessed from 400+ unique IPs in 30 countries.

**Initial Discovery**: The development cluster's API server was accessible from the internet. A Shodan scan had discovered it, and automated bots were probing the endpoint.

```bash
# What the attacker saw (and what the team saw in the logs)
curl -k https://k8s-api.dev.example.com:6443/api/v1/pods
# Response returned data — no authentication required
```

**Impact**: The API server was configured with `--anonymous-auth=true` and the RBAC permissions granted anonymous users access to cluster resources. For 3 hours, the cluster's resource metadata was exposed to the internet.

---

## Background

The development cluster was meant to be internal-only — accessible only via VPN. To simplify access for the team, the API server was exposed through a kube-nginx reverse proxy on a public DNS record.

The kube-nginx config:

```nginx
server {
    listen 6443 ssl;
    server_name k8s-api.dev.example.com;

    ssl_certificate /etc/nginx/certs/tls.crt;
    ssl_certificate_key /etc/nginx/certs/tls.key;

    location / {
        proxy_pass https://127.0.0.1:6443;
        proxy_set_header Host $host;
    }
}
```

No authentication at the nginx level. No IP whitelist. Just a straight pass-through to the API server.

The API server itself was started with:

```bash
kube-apiserver \
  --anonymous-auth=true \          # ← Anonymous requests allowed
  --authorization-mode=RBAC \       # RBAC enabled
  --allow-privileged=false
```

By default, Kubernetes has a `system:anonymous` user bound to the `system:discovery` ClusterRoleBinding — giving anonymous users access to API discovery endpoints:

```bash
# What anonymous users could access
kubectl --server=https://k8s-api.dev.example.com:6443 --insecure-skip-tls-verify api-resources
# API resources returned successfully
```

In this cluster, someone had also bound the `system:anonymous` user to the `view` ClusterRole (for monitoring purposes), granting read access to all resources:

```bash
# Someone created this binding for "monitoring"
kubectl create clusterrolebinding anonymous-view \
  --clusterrole=view \
  --user=system:anonymous
```

---

## Investigation

### Step 1: Confirm the Exposure

```bash
# From outside the cluster, check what's accessible
curl -sk https://k8s-api.dev.example.com:6443/api/v1/namespaces
```

The API returned a list of all namespaces. No authentication needed.

```bash
curl -sk https://k8s-api.dev.example.com:6443/api/v1/namespaces/production/secrets
# Returns 403 — secrets not accessible to anonymous (view role doesn't grant secret access)
```

### Step 2: Check Audit Logs

```bash
grep "system:anonymous" /var/log/kubernetes/audit.log | head -20
```

Audit logs showed 10,000+ anonymous API calls from 400+ IPs. Most were discovery probes, but several attempted:

- Listing pods across all namespaces
- Attempting to create pods
- Accessing ConfigMaps (which contained connection strings)
- Probing for privilege escalation via binding creation

### Step 3: Identify the Misconfiguration Chain

```bash
# Check ClusterRoleBindings involving anonymous
kubectl get clusterrolebinding -o json | jq -r '
  .items[] | select(.subjects[]?.name == "system:anonymous") |
  "\(.metadata.name) → \(.roleRef.name)"'
```

```
anonymous-view → view
system:discovery → system:discovery
```

The `anonymous-view` binding was the critical issue. Someone had added it months ago for a monitoring proof-of-concept and never removed it.

---

## Root Cause

1. **API server exposed to the internet**: The kube-nginx reverse proxy had no IP restrictions, no authentication, and was publicly accessible
2. **Anonymous auth enabled**: `--anonymous-auth=true` allowed unauthenticated requests to reach the RBAC layer
3. **Anonymous users granted view access**: The `anonymous-view` ClusterRoleBinding gave `system:anonymous` read access to all resources
4. **No network restriction**: No Cloudflare WAF, no VPN requirement, no IP allowlist at the nginx level
5. **No audit monitoring**: 10,000+ anonymous API calls without triggering any alert

---

## Resolution

### Emergency (Immediate)

```bash
# 1. Disable anonymous auth (temporarily)
# On the control plane node:
sed -i 's/--anonymous-auth=true/--anonymous-auth=false/' /etc/kubernetes/manifests/kube-apiserver.yaml
# kubelet will restart the API server automatically

# OR — remove the anonymous-view binding
kubectl delete clusterrolebinding anonymous-view

# Wait — with anonymous auth on but binding removed, anonymous users can only access discovery endpoints
```

Better approach: keep `--anonymous-auth=true` (needed for health checks and discovery) but remove the excessive bindings:

```bash
# 2. Remove the dangerous binding
kubectl delete clusterrolebinding anonymous-view

# 3. Restrict anonymous users to only discovery endpoints
# By default, the system:discovery ClusterRoleBinding only grants
# access to /api and /apis — this is safe and needed for cluster operation
```

### Add API Server Authentication

```bash
# 4. Add OIDC authentication
# Edit kube-apiserver.yaml to add OIDC flags
```

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --oidc-issuer-url=https://accounts.google.com
    - --oidc-client-id=xxxxx.apps.googleusercontent.com
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
```

### Secure the Reverse Proxy

```nginx
# Add IP whitelist and auth at nginx level
server {
    listen 6443 ssl;
    server_name k8s-api.dev.example.com;

    # IP whitelist — office IPs only
    allow 203.0.113.0/24;   # Office VPN
    allow 198.51.100.0/24;  # Remote office
    deny all;

    # Basic auth as additional layer
    auth_basic "Kubernetes API Server";
    auth_basic_user_file /etc/nginx/.htpasswd;

    ssl_certificate /etc/nginx/certs/tls.crt;
    ssl_certificate_key /etc/nginx/certs/tls.key;

    location / {
        proxy_pass https://127.0.0.1:6443;
    }
}
```

### Use Cloudflare or Similar for API Protection

```bash
# Route API traffic through Cloudflare
# Cloudflare → Authenticated Origin Pull → kube-nginx
```

Features:
- DDoS protection
- IP geolocation blocking
- Rate limiting
- Access (zero-trust authentication)

### Enable Audit Log Alerting

```yaml
# Falco rule: detect API calls from anonymous users
- rule: Anonymous API Access
  desc: Detect API requests from system:anonymous user
  condition: ka.user.name = "system:anonymous"
  output: "Anonymous API access detected (user=%ka.user.name verb=%ka.verb resource=%ka.target.resource)"
  priority: WARNING
```

```yaml
# Prometheus alert for anonymous API spike
- alert: AnonymousAPISpike
  expr: rate(apiserver_request_anonymous_total[5m]) > 10
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "High rate of anonymous API requests"
```

---

## Lessons Learned

- **API server should NEVER be directly internet-accessible**: Always use a VPN, Cloudflare Access, or at minimum IP whitelisting. The kube-nginx proxy is not a security boundary
- **`system:anonymous` is a real user**: Treat anonymous access like any other user. Audit what permissions anonymous has and revoke anything beyond discovery endpoints
- **Default anonymous permissions are safe — custom bindings are dangerous**: The default `system:discovery` binding is fine. Adding `system:anonymous` to `view`, `edit`, or `admin` is equivalent to making your cluster public-read
- **OIDC is the standard for API auth**: Move beyond static tokens and client certificates. OIDC provides short-lived tokens, group mapping, and integration with existing SSO
- **VPC/internal-only should not be the only defense**: "It's only accessible via VPN" is not a security policy — it's a hope. Defence in depth applies to network security too

---

## Summary

The attack chain:

```
Development cluster API server exposed via kube-nginx on public DNS
→ Shodan scans internet for open Kubernetes API endpoints
→ Finds k8s-api.dev.example.com:6443
→ curl /api/v1/pods returns data — no auth needed
→ 400+ automated scanners probe the endpoint in 3 hours
→ system:anonymous user had view ClusterRoleBinding
→ ConfigMaps with connection strings accessed
→ No alerting triggered for anonymous access
```

Discovery: Cloudflare anomaly alert. Remediation: 1 hour to lock down. Root fix: remove anonymous-view binding + IP whitelist + OIDC. The API server is the crown jewels — don't leave the door open for anyone to walk in.
