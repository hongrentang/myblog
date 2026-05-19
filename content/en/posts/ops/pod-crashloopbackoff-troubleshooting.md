---
title: "Pod CrashLoopBackOff Troubleshooting: A ConfigMap Mismatch That Took Down Order Service"
date: 2026-05-19
weight: 100000
slug: "pod-crashloopbackoff-troubleshooting"
tags: ["kubernetes", "troubleshooting"]
categories: ["K8S"]
description: "Full walkthrough of diagnosing Pod CrashLoopBackOff — from misdiagnosing OOM to finding a ConfigMap key mismatch"
keywords: "k8s pod crashloopbackoff, pod keeps restarting, kubernetes crashloopbackoff fix, configmap troubleshooting"
draft: false
featured: true
cover:
  image: "/images/crashloopbackoff-banner.svg"
  caption: "Pod CrashLoopBackOff Troubleshooting & Diagnostics"
---

# Pod CrashLoopBackOff Troubleshooting

## Common Search Queries

If you're landing here from a search, this post covers:

- k8s pod crashloopbackoff
- pod keeps restarting kubernetes
- kubernetes crashloopbackoff fix
- configmap key not found pod crash
- kubectl logs --previous crashloopbackoff

---

## The Incident

**Environment**: K8S v1.28, containerd 1.7, Calico CNI, 3 Master + 5 Worker.

**Time**: 01:20 AM, right after a routine config change was deployed.

**Symptoms**: Monitoring alerted — `payment-service` replicas dropped from 3 Available to 0. All pods in `CrashLoopBackOff`.

```bash
NAMESPACE     NAME                                   READY   STATUS              
production    payment-service-7b8f9d4c5f-abc12       0/1     CrashLoopBackOff    
production    payment-service-7b8f9d4c5f-def34       0/1     CrashLoopBackOff    
production    payment-service-7b8f9d4c5f-ghi56       0/1     CrashLoopBackOff    
```

**Impact**: Users could not place orders. The order entry point was completely down.

---

## Background

A developer requested a database connection string update. Our SRE edited the ConfigMap, changed a few keys, and ran `kubectl rollout restart` on the deployment.

Two minutes later, the alerts fired.

At 01:20 AM, my first thought was resource pressure — maybe the new config caused higher memory usage and the pods got OOMKilled.

---

## Investigation

### Step 1: Inspect the Pod

```bash
kubectl describe pod payment-service-7b8f9d4c5f-abc12 -n production
```

No OOMKilled. No Evicted. But the Restart Count was climbing fast.

```
Containers:
  payment-service:
    Container ID:   containerd://abc123
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
    Restart Count:  12
```

Exit Code 1 — not 137 (OOM). The process was terminating itself, not being killed by the system. Resource limits were off the table.

### Step 2: Read the Logs

For CrashLoopBackOff pods, you need `--previous` to see the last termination's output:

```bash
kubectl logs payment-service-7b8f9d4c5f-abc12 -n production --previous
```

```
2026-05-19T01:15:23.001Z  INFO  Configuration loaded
2026-05-19T01:15:23.005Z  INFO  Connecting to database...
2026-05-19T01:15:23.008Z  FATAL  environment variable "DB_HOST" is not set
```

Clear as day — the app couldn't find the `DB_HOST` environment variable.

### Step 3: Check the ConfigMap

```bash
kubectl get configmap payment-service-config -n production -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: production
data:
  DATABASE_HOST: "10.0.1.100"
  DATABASE_PORT: "3306"
  DATABASE_USER: "payment"
  DATABASE_PASS: "****"
```

The ConfigMap had `DATABASE_HOST`, but the application expected `DB_HOST`. A classic key name mismatch.

### Step 4: Verify the Deployment Reference

The deployment's env section:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: payment-service-config
        key: DB_HOST
```

The deployment referenced `DB_HOST`, but the ConfigMap no longer had that key. The app started, found an empty string for `DB_HOST`, and exited immediately.

---

## Root Cause

The change history:

1. A developer requested a database connection string update
2. The SRE renamed the key from `DB_HOST` to `DATABASE_HOST` while editing the ConfigMap, thinking it was a naming convention cleanup
3. The deployment's env reference was never updated to match
4. After restart, pods couldn't resolve `DB_HOST` and crashed

**Misdiagnosis**: I spent the first 5 minutes investigating OOM and resource pressure. One `kubectl logs --previous` would have saved that time entirely.

---

## Resolution

### Hotfix (3 minutes)

Rollback the deployment:

```bash
kubectl rollout undo deployment/payment-service -n production
```

Pods recovered with the old config. Service restored.

### Proper Fix

Revert the ConfigMap key to match the deployment's expectation:

```yaml
data:
  DB_HOST: "10.0.1.100"          # Changed back to DB_HOST
  DATABASE_PORT: "3306"
  DATABASE_USER: "payment"
  DATABASE_PASS: "****"
```

Alternatively, update the deployment's env reference to point to `DATABASE_HOST`. Either approach works — the key is consistency.

### Long-Term Prevention

1. **ConfigMap changes must be reviewed**: No direct edits, use PR-based workflows
2. **Pre-apply validation**: Use `kubectl diff` or `helm diff` to catch config drift before rollout
3. **Liveness/Readiness checks**: Add startup probes to validate critical config

```yaml
readinessProbe:
  exec:
    command:
      - sh
      - -c
      - "[ -n \"$DB_HOST\" ] && [ -n \"$DB_PORT\" ]"
  initialDelaySeconds: 5
  periodSeconds: 5
```

---

## Lessons Learned

- **Exit Code 1 vs 137**: Exit Code 1 means the process chose to exit — check logs first. Exit Code 137 (SIGKILL) means OOM, check resource limits
- **Always use `kubectl logs --previous`** for CrashLoopBackOff — without it you're reading an empty container's output
- **ConfigMap key changes don't trigger rolling updates**: You must `rollout restart`, but if the key name changed, pods will crash on restart
- **Don't rename config keys during a change**: If it's not broken, don't fix it. Renaming is a separate task that needs dev confirmation

---

## Summary

The investigation chain:

```
Alert → Pod CrashLoopBackOff
  → Misdiagnosed OOM (checked resource metrics)
  → Saw Exit Code 1 (ruled out OOM)
  → Checked logs (kubectl logs --previous)
  → Found DB_HOST not set
  → Compared ConfigMap (key name mismatch)
  → Rollback → Fixed ConfigMap → Postmortem
```

Total time from alert to recovery: 8 minutes. Five of those were wasted on the wrong hypothesis.

**First rule of debugging: check the logs, don't guess.**
