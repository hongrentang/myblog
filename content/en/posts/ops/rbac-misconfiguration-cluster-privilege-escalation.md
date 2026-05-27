---
title: "RBAC Misconfiguration — How a Read-Only ServiceAccount Took Over the Cluster"
date: 2026-05-27
weight: 100190
slug: "rbac-misconfiguration-cluster-privilege-escalation"
tags: ["kubernetes", "security", "rbac", "troubleshooting"]
categories: ["Security"]
description: "A Kubernetes RBAC misconfiguration incident — how a ServiceAccount meant for read-only monitoring was accidentally granted cluster-admin, and how to audit permissions before it's too late"
keywords: "kubernetes rbac misconfiguration, k8s serviceaccount permissions, clusterrolebinding security, kubernetes privilege escalation, kubectl auth can-i"
draft: false
featured: true
cover:
  image: ""
  caption: "RBAC Misconfiguration — Privilege Escalation"
---

# RBAC Misconfiguration — How a Read-Only ServiceAccount Took Over the Cluster

## Common Search Queries

- kubernetes rbac misconfiguration example
- k8s serviceaccount has too many permissions
- clusterrolebinding security incident
- how to audit kubernetes rbac permissions
- kubectl auth can-i check permissions

---

## The Incident

**Environment**: K8S v1.27, kubeadm, 3 Master + 7 Worker, 200+ namespaces.

**Time**: 14:30, mid-day peak traffic.

**Symptoms**: A monitoring ServiceAccount (`monitoring/metrics-collector`) somehow deleted 15 production pods across 5 namespaces. The monitoring pipeline went down, and along with it, several critical services.

```bash
kubectl get events --all-namespaces --field-selector reason=Killing
NAMESPACE      LAST SEEN   TYPE    REASON   OBJECT                  MESSAGE
api-prod       14:29:53    Normal  Killing  pod/api-v2-7d9f8c6b-*   Stopping container api-server
api-prod       14:29:54    Normal  Killing  pod/api-v2-7d9f8c6b-*   Stopping container sidecar
cache-prod     14:29:55    Normal  Killing  pod/redis-sentinel-*    Stopping container redis
...
```

**Impact**: 15 pods deleted. 6 minutes of partial service degradation until the workloads were rescheduled.

---

## Background

Three weeks earlier, a new monitoring stack was deployed. The DevOps engineer needed a ServiceAccount that could list pods and read metrics. They created a Role with the following permissions:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: monitoring
  name: metrics-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
```

And then bound it... incorrectly:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: metrics-reader    # ← Can't find a ClusterRole named "metrics-reader" →
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
```

Wait — `metrics-reader` was defined as a **Role** (namespaced), not a **ClusterRole** (cluster-scoped). The binding referred to a ClusterRole named `metrics-reader` that didn't exist at the time. So the binding failed silently? No — Kubernetes doesn't validate RoleRef at bind time. It just stores the reference.

Meanwhile, the engineer noticed the ServiceAccount couldn't list pods. So they thought: "Let me just use a built-in ClusterRole that can read things." They found `view` — a built-in ClusterRole that grants read-only access to most resources.

But in the process of editing the YAML, they made two mistakes:

1. They changed the `roleRef` to `cluster-admin` to test if it would work (and forgot to change it back)
2. The `metrics-reader` Role was never cleaned up

The final binding:

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # ← Yikes
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
```

The monitoring ServiceAccount now had **cluster-admin** privileges across the entire cluster.

---

## Investigation

### Step 1: Identify Who Did What

```bash
# Check audit logs for the pod deletion events
grep "metrics-collector" /var/log/kubernetes/audit.log | grep "delete" | head -5
```

```
{"kind":"Event","level":"RequestResponse","user":{"username":"system:serviceaccount:monitoring:metrics-collector",...},"verb":"delete","objectRef":{"resource":"pods","namespace":"api-prod","name":"api-v2-7d9f8c6b-abc"}}
```

The ServiceAccount `monitoring/metrics-collector` was the actor. Every deleted pod traced back to it.

### Step 2: Check the ServiceAccount's Effective Permissions

```bash
kubectl auth can-i --list --as=system:serviceaccount:monitoring:metrics-collector
```

```
Resources                                       Non-Resource URLs   Verbs
*.*                                             []                  [*]
[*]                                             []                  [*]
```

This ServiceAccount had wildcard access to everything. That was the moment we knew: someone accidentally bound `cluster-admin` to a monitoring account.

### Step 3: Find the Binding

```bash
kubectl get clusterrolebinding monitoring-metrics-reader -o yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin    # ← Root cause right here
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
```

### Step 4: Trace Who Made the Change

```bash
# Check Kubernetes audit logs for who created/modified this binding
grep "monitoring-metrics-reader" /var/log/kubernetes/audit.log | head -3
```

```
...
"user":{"username":"devops-user"},"verb":"create","objectRef":{"resource":"clusterrolebindings","name":"monitoring-metrics-reader"},"responseStatus":{"code":201}}
...
"user":{"username":"devops-user"},"verb":"update","objectRef":{"resource":"clusterrolebindings","name":"monitoring-metrics-reader"},"responseStatus":{"code":200}}
```

The binding was created as `cluster-admin` from the start — this wasn't escalated by an attacker. It was a configuration error made during setup.

---

## Root Cause

The DevOps engineer used `cluster-admin` as a temporary test during YAML editing, and the file was applied before they could change it back. The monitoring stack was deployed under time pressure, and the extra permissions went unnoticed because:

- Nobody reviewed the RBAC manifests before applying them
- No automated RBAC policy scanning was in place
- The monitoring system worked, so no one looked deeper
- No alerting was set up for high-privilege binding creation

---

## Resolution

### Emergency (Immediate)

```bash
# Fix the ClusterRoleBinding to use the correct role
kubectl edit clusterrolebinding monitoring-metrics-reader
```

Change `cluster-admin` to `view`:

```yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view    # Fixed: read-only access
```

**Note**: `roleRef` in a binding is immutable — you need to delete and recreate, or create a new binding:

```bash
kubectl delete clusterrolebinding monitoring-metrics-reader

cat <<EOF | kubectl apply -f -
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: monitoring-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: metrics-collector
  namespace: monitoring
EOF
```

### Audit All Bindings

```bash
# List all ClusterRoleBindings and their roles
kubectl get clusterrolebinding -o custom-columns='NAME:.metadata.name,ROLE:.roleRef.name,KIND:.roleRef.kind,SUBJECTS:.subjects[*].name' | sort
```

Check for any ServiceAccount bound to `cluster-admin`:

```bash
kubectl get clusterrolebinding -o json | jq -r '
  .items[] | select(.roleRef.name == "cluster-admin") |
  "\(.metadata.name) → \(.subjects[]?.kind)/\(.subjects[]?.name) [\(.subjects[]?.namespace // "cluster")]"
'
```

### RBAC Best Practices Applied

1. **Principle of Least Privilege**: Never start with `cluster-admin` and reduce — start with minimum permissions and expand only when proven necessary
2. **Dedicated Roles per Use Case**: Don't reuse shared ClusterRoles like `admin`, `edit`, `view` for specific needs — create custom Roles
3. **Review RBAC Manifests in CI**: Use tools like `kube-linter`, `checkov`, or `kube-bench` to scan for over-privileged bindings before apply
4. **Audit Alerting**: Set up alerts for ClusterRoleBinding creation with sensitive roles

```yaml
# Example: Prometheus rule for RBAC alerting
groups:
- name: rbac-security
  rules:
  - alert: HighPrivilegeBindingCreated
    expr: count(kube_clusterrolebinding_info{role="cluster-admin"}) > 3
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "New cluster-admin binding detected"
```

---

## Lessons Learned

- **`roleRef` is case-sensitive and type-sensitive**: Binding to a `Role` vs `ClusterRole` matters. The wrong type can silently fail to bind, leading to "just make it work" escalations
- **`cluster-admin` is never a temporary test**: It sticks. Always use a dedicated test namespace with a restricted Role
- **Monitor who can do what**: `kubectl auth can-i --list --as=<user>` reveals effective permissions. Run it for every critical ServiceAccount after setup
- **RBAC does not validate RoleRef at creation time**: You can bind to a ClusterRole that doesn't exist, and it will silently apply no permissions — until someone creates that ClusterRole later

---

## Summary

The investigation chain:

```
Monitoring pipeline reports permission denied
→ DevOps engineer tests with cluster-admin "temporarily"
→ Applies YAML before remembering to revert
→ ServiceAccount now has cluster-wide admin access
→ No code review, no RBAC scan catches it
→ Weeks later, a misconfigured script in the monitoring container calls kubectl delete pods
→ 15 pods deleted, partial outage for 6 minutes
```

Total time to remediate: 15 minutes. Total time the over-privileged binding existed: 3 weeks. The fix was simple — the cost was a production incident that should never have happened.

Audit your RBAC. Before the incident, not after.
