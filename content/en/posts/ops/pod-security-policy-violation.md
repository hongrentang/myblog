---
title: "Pod Security Policy Violation — How a Privilege Escalation Pod Bypassed All Cluster Defenses"
date: 2026-06-02
weight: 100270
slug: "pod-security-policy-violation"
tags: ["kubernetes", "security", "pod-security", "psa", "troubleshooting"]
categories: ["Security"]
description: "A Pod Security Standards incident — how a pod with privilege escalation and host paths bypassed namespace restrictions, and why PSA with enforcement is necessary from Day 1"
keywords: "kubernetes pod security policy, psa admission, privilege escalation container, allowPrivilegeEscalation, pod security standards enforcement"
draft: false
featured: true
cover:
  image: ""
  caption: "Pod Security Violation — Incident Response"
---

# Pod Security Policy Violation — How a Privilege Escalation Pod Bypassed All Cluster Defenses

## Common Search Queries

- kubernetes pod security admission violation
- privilege escalation container pod security
- allowPrivilegeEscalation true security risk
- pod security standards enforced vs warn
- kubernetes psa audit mode bypass

---

## The Incident

**Environment**: K8S v1.28, 10 clusters, Pod Security Admission (PSA) configured in `warn` mode only. No enforcement.

**Time**: Thursday 09:00. Security audit report flagged a container running with `allowPrivilegeEscalation: true` and host network access in the production namespace.

**Initial Discovery**: A quarterly compliance scan using `kube-bench` and `kubescape` found multiple pods violating Pod Security Standards in supposedly restricted namespaces.

```bash
kubescape scan --enable-host-scan -n production
# Results:
# ❌ AllowPrivilegeEscalation: container "worker" in pod "data-processor-7d9f8c6b-x"
# ❌ HostNetwork: container "worker" in pod "data-processor-7d9f8c6b-x"
# ❌ RunAsRoot: container "worker" running as UID 0
```

**Impact**: 12 pods across 3 production namespaces were running with excessive privileges that violated Pod Security Standards. No active exploitation was detected — but the exposure had existed for 6 months.

---

## Background

When Pod Security Admission (PSA) was introduced in Kubernetes 1.23, the team knew they needed to adopt it. They configured PSA with `warn` mode to avoid breaking existing workloads:

```yaml
# Namespace labels — warn mode only, no enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/warn: restricted       # Only warns
    pod-security.kubernetes.io/audit: restricted       # Logs violations
    # Note: no "enforce" label — violations allowed
```

This meant PSA would **warn** about violations but **never block** them. The namespace had a "restricted" profile in name only.

Over 6 months, several data processing pods were deployed with escalating privileges. Each time, the developers saw the PSA warning in the pod description but ignored it because:

```bash
# PSA warning when creating the pod
Warning: would violate PodSecurity "restricted:latest": privilege escalation
  (containers with "allowPrivilegeEscalation=true" or "privileged=true")
```

"Just a warning. It deployed fine. We'll fix it later." — Developer comment in the ticket that was never resolved.

The problematic pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: data-processor
  namespace: production
spec:
  containers:
  - name: worker
    image: data-processor:latest
    securityContext:
      allowPrivilegeEscalation: true    # ← PSA restricted violation
      runAsUser: 0                       # ← Running as root
      capabilities:
        add: ["SYS_ADMIN", "NET_ADMIN"] # ← Excessive capabilities
    ports:
    - containerPort: 80
    hostNetwork: true                    # ← Host network access
```

---

## Investigation

### Step 1: Audit All Namespaces for PSA Violations

```bash
# Check PSA mode on all namespaces
kubectl get ns -o json | jq -r '
  .items[] | "\(.metadata.name): enforce=\(.metadata.annotations["pod-security.kubernetes.io/enforce"] // "none") warn=\(.metadata.annotations["pod-security.kubernetes.io/warn"] // "none")"'
```

```
production: enforce=none warn=restricted
staging: enforce=none warn=baseline
development: enforce=none warn=baseline
critical-prod: enforce=none warn=restricted
```

Zero namespaces with `enforce` mode. PSA was advisory only across the entire cluster.

### Step 2: Identify All Violating Pods

```bash
# Check all pods for privilege escalation
kubectl get pods --all-namespaces -o json | jq -r '
  .items[] | select(.spec.containers[].securityContext.allowPrivilegeEscalation == true or .spec.containers[].securityContext.privileged == true) |
  "\(.metadata.namespace)/\(.metadata.name): escalation=\(.spec.containers[].securityContext.allowPrivilegeEscalation)"'
```

12 pods found violating PSA restricted profile:
- 6 with `allowPrivilegeEscalation: true`
- 3 running as root (`runAsUser: 0`)
- 3 with host network access
- 2 with privileged mode (some overlapped)

### Step 3: Check for Active Exploitation

```bash
# Check audit logs for suspicious activity from these pods
grep -f /tmp/violating-pod-ips /var/log/kubernetes/audit.log | grep -E "delete|create|exec|secret" | head -20
```

No signs of active exploitation. These were legitimate workloads with excessive permissions — security debt, not an active breach.

---

## Root Cause

1. **PSA in warn-only mode**: The `enforce` label was never set on any namespace. PSA was a "please consider" sign, not a security control
2. **No admission controller blocking violations**: No OPA/Gatekeeper or Kyverno policy was in place to enforce Pod Security Standards
3. **Security warnings treated as optional**: Developers saw PSA warnings and ignored them. Without enforcement, warnings have no teeth
4. **No compliance automation**: The violations existed for 6 months without being flagged by any automated scan
5. **Privileged workloads in production**: Data processing workloads with host access and privilege escalation don't belong in the production namespace with restricted labeling

---

## Resolution

### Enable PSA Enforcement

```bash
# Apply enforce mode to all namespaces (carefully)
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite
```

This immediately blocks any new pod that violates the restricted profile. Existing violating pods continue running but cannot be updated without compliance:

```bash
# Check enforcement status
kubectl describe namespace production | grep -A5 "Pod Security"
```

### Remediate Existing Violations

For each violating pod, work with the application team to:

1. Remove `allowPrivilegeEscalation: true` where possible
2. Switch to non-root user via `runAsNonRoot: true`
3. Remove `hostNetwork: true` — use port forwarding or NodePort instead
4. Drop unnecessary capabilities, keep only `NET_BIND_SERVICE` if needed

```yaml
# Remediated pod spec
securityContext:
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 1001
  capabilities:
    drop: ["ALL"]
    add: ["NET_BIND_SERVICE"]
  seccompProfile:
    type: RuntimeDefault
```

### Use Exemptions When Necessary

For workloads that genuinely need elevated privileges (e.g., node monitoring, network plugins):

```yaml
# Use namespace exemptions for specific system workloads
apiVersion: v1
kind: Namespace
metadata:
  name: system-monitoring
  labels:
    pod-security.kubernetes.io/enforce: privileged   # Full privileges allowed
    pod-security.kubernetes.io/enforce-version: latest
```

Or use OPA/Gatekeeper exemptions instead of weakening namespace PSA:

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSAExemptions
metadata:
  name: exempt-monitoring-agents
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    exemptionImages:
      - "prometheus/node-exporter:*"
      - "fluent/fluentd:*"
```

### Automate Compliance Scanning

```bash
# Add kubescape to CI/CD
kubescape scan framework nsa --exceptions exceptions.json

# Fail build on critical violations
kubescape scan --fail-threshold critical
```

### Monitoring

```yaml
# Alert on PSA enforce failures
- alert: PodSecurityViolationBlocked
  expr: increase(kubernetes_feature_pod_security_enforce_errors_total[5m]) > 0
  for: 1m
  labels:
    severity: warning
  annotations:
    summary: "Pod Security Standards are blocking workloads"
```

---

## Lessons Learned

- **PSA `warn` is not security**: Warning mode logs violations but does nothing about them. Without `enforce`, PSA is a suggestion, not a policy
- **Security warnings become noise**: Developers see hundreds of warnings daily. PSA warnings without consequences are ignored within weeks
- **PSA alone is not enough**: Combine PSA with OPA/Gatekeeper or Kyverno for policies beyond the three built-in profiles
- **Progressive enforcement works**: Start with `audit`, move to `warn`, then `enforce`. But don't stop at `warn` — that's not the finish line
- **System workloads need exceptions**: Some system components (node exporters, network plugins) legitimately need higher privileges. Use dedicated namespaces labels for them

---

## Summary

The security gap:

```
Team adopts PSA in warn mode to avoid breaking workloads
→ Namespaces labeled with "warn: restricted" but NOT "enforce: restricted"
→ Developers deploy pods with privilege escalation, root access, host network
→ PSA shows warnings — nobody acts on them
→ 12 violating pods accumulate over 6 months
→ Compliance scan discovers violations during quarterly audit
→ PSA enforce mode enabled, pods remediated
```

The fix: change one label from `warn` to `enforce`. The work: remediating 6 months of security debt. PSA enforce from Day 1 would have cost nothing. PSA warn-only cost 6 months of exposure and a major remediation effort. Enforce early, enforce often.
