---
title: "ArgoCD Sync Failure — When GitOps Refused to Deploy and Nobody Knew Why"
date: 2026-06-10
weight: 100490
slug: "argocd-sync-failure-deployment-stuck"
tags: ["argocd", "gitops", "ci-cd", "kubernetes", "troubleshooting"]
categories: ["Troubleshooting"]
description: "An ArgoCD sync failure incident — how a mismatched server-side apply field manager and a corrupted ConfigMap hash caused all application syncs to fail with 'comparison failed' errors, blocking all deployments"
keywords: "argocd sync failure, argocd comparison failed, gitops troubleshooting, argocd out of sync, argocd application stuck"
draft: false
featured: true
cover:
  image: ""
  caption: "ArgoCD Sync Failure — Troubleshooting"
---

## Common Search Queries

- ArgoCD sync failed comparison failed
- ArgoCD OutOfSync stuck fix
- ArgoCD last-applied-configuration annotation too large
- ArgoCD server-side apply field manager conflict
- ArgoCD hard refresh vs soft refresh
- ArgoCD repo server cache stale
- argocd app list all OutOfSync

## 故障经过 / The Incident

### Environment

- **ArgoCD Version**: v2.10
- **Managed Clusters**: 3 Kubernetes clusters (production, staging, DR)
- **Application Count**: 50+ applications managed via ArgoCD
- **Git Repository**: GitLab (on-premise)
- **Deployment Method**: Helm charts with Kustomize overlays
- **Cluster Size**: ~200 nodes total, ~3000 pods

### Timeline

It was a routine afternoon deploy. A developer pushed a Helm values update to GitLab — a simple ConfigMap change for a logging configuration. The git push succeeded without errors. CI pipeline passed. Then the production team noticed something was wrong.

**14:30 UTC** — Git push to main branch with ConfigMap update.

**14:32 UTC** — CI pipeline completes successfully, artifact published.

**14:35 UTC** — Platform team reports: all 50+ applications in ArgoCD UI show `OutOfSync` status. Sync buttons produce errors.

**14:40 UTC** — Emergency response initiated.

### Symptoms

- **ArgoCD Web UI**: Every application displays "OutOfSync" status with "Sync Status: Failed" in red
- **CLI output**: `argocd app list` returns all applications with `STATUS=OutOfSync`, `HEALTH=Degraded` or `HEALTH=Missing`
- **Sync attempt**: Clicking "Sync" or running `argocd app sync <app>` produces `comparison failed` error immediately
- **No actual change**: Even applications whose git content had NOT changed were failing
- **Partial output**: Some application manifests appeared truncated or malformed in the ArgoCD UI diff view
- **Git is fine**: GitLab shows the push succeeded, CI passed, manifests rendered correctly

The most alarming symptom was that this was not a single-application failure — it was a total platform outage. ArgoCD was refusing to sync anything.

## Background / 背景

To understand why this happened, we need to review how ArgoCD works under the hood.

### ArgoCD Architecture

ArgoCD has three core components:

1. **API Server** — Exposes the ArgoCD API and Web UI. Handles app management, authentication, and RBAC.
2. **Repo Server** — Clones git repositories, generates manifests (Helm template, Kustomize build, etc.), and caches the results. This is where manifest generation happens.
3. **Application Controller** — The heart of ArgoCD. Runs the reconciliation loop: compares desired state (from git) against live state (in the cluster), detects drift, and triggers sync operations.

### Sync Mechanism

The ArgoCD reconciliation loop works in three phases:

1. **Diff** — The Repo Server generates the desired manifests from git. The Application Controller compares these against the live cluster state using a three-way diff (current live state vs desired state vs last-applied-configuration annotation).
2. **Apply** — If drift is detected, ArgoCD applies the desired state to the cluster. This can use either client-side apply (default, uses `kubectl apply` logic via the `last-applied-configuration` annotation) or server-side apply (uses SSA field management).
3. **Health Check** — After sync, ArgoCD evaluates resource health based on built-in or custom health checks.

### Server-Side Apply vs Client-Side Apply

- **Client-Side Apply (CSA)**: The traditional approach. ArgoCD stores the last applied manifest in the `kubectl.kubernetes.io/last-applied-configuration` annotation. On each sync, it computes a strategic merge patch between live state, desired state, and the annotation. This annotation can grow very large over time.
- **Server-Side Apply (SSA)**: A newer approach introduced in Kubernetes 1.18+. Instead of storing state in an annotation, SSA uses `fieldManager` ownership tracked on each field in the live object. ArgoCD uses `argocd-controller` as its field manager by default. SSA avoids the annotation bloat problem but introduces field ownership conflicts when multiple tools manage the same fields.

### Resource Tracking

ArgoCD tracks the resources it manages using a combination of labels and annotations:

- `app.kubernetes.io/instance` — label identifying which ArgoCD Application manages this resource
- `argocd.argoproj.io/tracking-id` — annotation with detailed tracking info

These tracking labels allow ArgoCD to map live cluster resources back to Application definitions, even when resource names change.

## 排查过程 / Investigation

### Step 1: Check App Status — All Applications Degraded

The first command confirmed the worst:

```bash
argocd app list
```

Output showed every application with:
```
NAME              CLUSTER  NAMESPACE  PROJECT  STATUS     HEALTH       SYNCPOLICY  CONDITIONS
app-audit         prod-c1  default    default  OutOfSync  Degraded     <none>      ComparisonError
app-billing       prod-c1  default    default  OutOfSync  Missing      <none>      ComparisonError
app-cache         prod-c1  default    default  OutOfSync  Degraded     <none>      ComparisonError
...
```

50+ applications, all with `STATUS=OutOfSync` and `CONDITIONS=ComparisonError`. This was not an application-level issue — something was broken at the platform level.

### Step 2: Get Detailed Status — Comparison Failed

```bash
argocd app get app-audit
```

The detailed output revealed the nature of the error:

```
NAME:       app-audit
PROJECT:    default
SERVER:     https://kubernetes.default.svc
NAMESPACE:  default
URL:        https://argocd.example.com/applications/app-audit
REPO:       https://gitlab.example.com/infra/gitops-manifests
TARGET:     main
PATH:       apps/audit
SYNC STATUS:  OutOfSync (comparison failed)
HEALTH STATUS: Degraded

CONDITION:
  (ComparisonError)  comparison failed
```

The phrase `comparison failed` was the key. It meant ArgoCD could not compute the diff between desired and live state. This is a fundamental failure — without a valid diff, ArgoCD cannot determine what to change.

### Step 3: Hard Refresh

```bash
argocd app get app-audit --hard-refresh
```

A hard refresh forces ArgoCD to:
1. Re-clone the git repository (bypassing the Repo Server cache)
2. Re-generate manifests from scratch
3. Re-fetch live cluster state

The `--hard-refresh` flag took significantly longer than usual (about 90 seconds vs the normal 5-10 seconds). This was our first clue that the Repo Server or the controller was struggling with something.

After the hard refresh completed, SOME applications recovered temporarily — but within minutes they fell back to `OutOfSync` and `comparison failed`. This suggested a cache invalidation issue that was only temporarily resolved by the hard refresh.

### Step 4: Check ArgoCD Logs

```bash
kubectl logs -n argocd deployment/argocd-application-controller --tail=200
```

The Application Controller logs showed repeating error patterns:

```
time="14:35:01" level=error msg="comparison failed" app=app-audit
  error="failed to calculate diff: error getting live state: 
  unexpected end of JSON input"

time="14:35:02" level=error msg="comparison failed" app=app-billing
  error="failed to calculate diff: error getting live state: 
  unexpected end of JSON input"
```

The `unexpected end of JSON input` error hinted at a corrupted or truncated resource object. The Application Controller reads live state from the Kubernetes API, converts it to JSON, and compares it against the desired state. If a resource object has a malformed annotation (like a truncated JSON blob in `last-applied-configuration`), the JSON parser fails.

```bash
kubectl logs -n argocd deployment/argocd-repo-server --tail=200
```

The Repo Server logs showed:

```
time="14:35:01" level=warning msg="cache miss for revision abc123def"
time="14:35:02" level=error msg="failed to generate manifest from 
  repo gitlab.example.com/infra/gitops-manifests: 
  manifest generation error (cached): helm template failed: 
  error running helm: signal: killed"
```

The `signal: killed` message for `helm template` suggested the Repo Server was running out of memory during manifest generation. This was a side effect — the Repo Server's cache had grown too large because of the oversized annotations being fetched during resource comparison.

### Step 5: Check Git Connection

```bash
argocd repo list
```

The repo list showed all repositories as accessible:

```
SERVER                          STATUS  MESSAGE
https://gitlab.example.com/infra/gitops-manifests  Successful
https://gitlab.example.com/infra/platform-base     Successful
```

Git connectivity was fine. The issue was not a network problem.

### Step 6: Diff a Specific Resource

```bash
argocd app diff app-audit --local-path /tmp/manifests
```

This command generates the manifests locally and compares them against the live cluster state. The output showed:

```
===== configmap/log-config ======
-  - name: LOG_LEVEL
-    value: "debug"
+  - name: LOG_LEVEL
+    value: "info"
--- last-applied-configuration
-  <OMITTED, TOO LARGE>
```

The `OMITTED, TOO LARGE` marker was the smoking gun. ArgoCD was unable to read the `last-applied-configuration` annotation because it was too large to process.

### Step 7: Check Resource Annotations

```bash
kubectl get configmap log-config -n logging -o yaml | grep -A 20 "annotations"
```

The output revealed the problem:

```yaml
annotations:
  kubectl.kubernetes.io/last-applied-configuration: |
    {"apiVersion":"v1","kind":"ConfigMap","metadata":{...},"data":{...}}
    [TRUNCATED IN KUBECTL OUTPUT — SIZE EXCEEDS 256KB]
  argocd.argoproj.io/tracking-id: "app-audit:configmap:logging/log-config"
```

The `last-applied-configuration` annotation was enormous. Let me explain why.

### Step 8: Check Annotation Size

```bash
kubectl get configmap log-config -n logging -o json | jq '.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]' | wc -c
```

```
283472
```

**283KB**. The annotation was 283KB in size. Kubernetes stores annotations as plain text with a maximum size limit of 256KB for the total annotations metadata (though in practice, individual annotation values can cause issues well before hitting the hard limit). ArgoCD's diff engine, which reads this annotation to compute the three-way merge patch, was choking on the oversized data.

But how did it get this large? The ConfigMap contained rendered Helm templates with all their values expanded — logging configuration, parsing rules, output definitions, and multi-line regex patterns. Each deploy appended more data, and the annotation grew without bound.

### Step 9: Check Server-Side Apply Field Manager Conflicts

```bash
kubectl describe configmap log-config -n logging
```

Look at the `managedFields` section:

```yaml
managedFields:
- manager: argocd-controller
  operation: Apply
  apiVersion: v1
  fieldsType: FieldsV1
  fieldsV1:
    f:data: {}
    f:metadata:
      f:annotations:
        .: {}
        f:argocd.argoproj.io/tracking-id: {}
- manager: helm
  operation: Update
  apiVersion: v1
  fieldsType: FieldsV1
  fieldsV1:
    f:data:
      f:LOG_LEVEL: {}
    f:metadata:
      f:annotations:
        f:kubectl.kubernetes.io/last-applied-configuration: {}
```

There were TWO field managers operating on the same ConfigMap:

1. **argocd-controller** — using `Apply` (server-side apply)
2. **helm** — using `Update` (client-side apply, via `helm upgrade --force`)

The Helm `--force` operation had recreated the resource, resetting the SSA field ownership and leaving orphaned entries. When ArgoCD tried to sync next, it detected a field ownership conflict: the `data` field was claimed by both `argocd-controller` and `helm`, but with different operation types (`Apply` vs `Update`). This conflict prevented ArgoCD from cleanly applying its desired state.

## 根因 / Root Cause

After piecing together all the evidence, the root cause analysis identified three interconnected issues:

### Primary Cause: Oversized last-applied-configuration Annotation

A single ConfigMap (`log-config`) had accumulated a `kubectl.kubernetes.io/last-applied-configuration` annotation of 283KB. This annotation contained the full rendered Helm template output — every ConfigMap key, every log parsing rule, every multi-line regex pattern — serialized as a single JSON string.

ArgoCD's diff engine reads this annotation for every reconciliation cycle to compute the three-way strategic merge patch (live vs desired vs last-applied). When the annotation exceeded ~200KB, the JSON parser in the Application Controller began failing with `unexpected end of JSON input` errors. This caused the `comparison failed` status for every Application that referenced this ConfigMap.

### Secondary Cause: Server-Side Apply Field Manager Conflict

The ConfigMap had been modified by multiple tools:

- ArgoCD (via SSA, field manager: `argocd-controller`, operation: `Apply`)
- Helm (via `helm upgrade --force`, which does a client-side Update that orphans SSA field ownership)

This created a situation where ArgoCD could not determine which field manager owned the `data` section. The `helm --force` operation effectively performed a delete-and-recreate, which reset the SSA managed fields and left the `argocd-controller` field manager in an inconsistent state.

### Tertiary Cause: Repo Server Cache Staleness

The Repo Server caches generated manifests to avoid re-running `helm template` or `kustomize build` on every reconciliation. Under normal operation, this cache is a performance optimization. However, in this incident:

1. The oversized annotation caused repeated failure in the controller
2. The controller retried continuously, generating load on the Repo Server
3. The Repo Server's memory grew as it cached corrupted or partial results
4. The cache became poisoned — returning stale or incomplete manifest data
5. Even after the annotation was cleaned, the stale cache caused continued failures until a hard refresh or restart cleared it

### Why All Applications Failed Simultaneously

This was the most confusing aspect of the incident. Why did ALL 50+ applications fail when only one ConfigMap had the oversized annotation?

The answer lies in the **SharedInformer cache**. ArgoCD's Application Controller uses a SharedInformer to watch resources across all managed clusters. When the informer received the malformed ConfigMap object (with the oversized annotation), it attempted to cache it in memory. The corrupted JSON in the annotation caused the informer's internal cache to become inconsistent. Since the SharedInformer is shared across all Applications watching that cluster, a single corrupted object poisoned the cache for every Application.

Additionally, the Repo Server's manifest cache was shared across applications that used the same git repository. When one application's manifest generation hit an error, the cached error was returned to other applications using the same repo, causing cascading failures.

## 解决方案 / Resolution

### Emergency Fix (Immediate)

We executed the following steps to restore service:

#### Step 1: Remove the Corrupted Annotation

First, we identified and removed the oversized annotation from the problematic ConfigMap:

```bash
kubectl annotate configmap log-config -n logging \
  kubectl.kubernetes.io/last-applied-configuration-
```

The trailing `-` tells kubectl to REMOVE the annotation. This immediately freed the Application Controller from trying to parse the 283KB JSON blob.

**Verification**:

```bash
kubectl get configmap log-config -n logging -o json | \
  jq '.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]'
# Output: null — annotation removed
```

#### Step 2: Hard Refresh All Applications

Hard refresh forces ArgoCD to re-clone repos and re-fetch live state from the cluster:

```bash
argocd app list -o name | xargs -I {} argocd app get {} --hard-refresh
```

For a more targeted approach, we refreshed only the applications referencing the affected namespace:

```bash
argocd app list --selector "cluster=prod-c1" -o name | \
  xargs -I {} argocd app get {} --hard-refresh
```

#### Step 3: Fix SSA Field Manager Conflicts

For resources with SSA conflicts, we re-applied the manifest using `--force-conflicts` to establish ArgoCD as the authoritative field manager:

```bash
kubectl apply --server-side --force-conflicts \
  --field-manager=argocd-controller \
  -f corrected-manifest.yaml -n logging
```

This told Kubernetes: "I know there are conflicts, force ArgoCD's version of the fields."

#### Step 4: Restart Repo Server

To clear the poisoned manifest cache:

```bash
kubectl rollout restart deployment/argocd-repo-server -n argocd
```

Wait for the rollout to complete:

```bash
kubectl rollout status deployment/argocd-repo-server -n argocd --timeout=300s
```

After restart, the Repo Server started with a clean cache.

#### Step 5: Sync Applications

Finally, trigger a sync for the affected applications:

```bash
argocd app sync log-config --prune --timeout 300
```

For bulk sync:

```bash
argocd app list -o name | xargs -I {} argocd app sync {} --prune --timeout 300
```

Monitor the sync status:

```bash
argocd app list --watch
```

**Service restored at 15:20 UTC.** Total downtime: approximately 50 minutes.

### Long-Term Preventive Measures

After restoring service, we implemented the following improvements:

#### 1. ConfigMap Size Linting in CI/CD

Added a CI pipeline check that warns if any generated manifest exceeds 100KB or if the last-applied-configuration annotation would exceed 50KB:

```yaml
# .gitlab-ci.yml snippet
check-manifest-size:
  stage: validate
  script:
    - kubectl create configmap test --from-file=manifests/ --dry-run=client -o yaml > /tmp/manifest-check.yaml
    - MANIFEST_SIZE=$(wc -c < /tmp/manifest-check.yaml)
    - if [ "$MANIFEST_SIZE" -gt 102400 ]; then echo "WARNING: Manifest exceeds 100KB ($MANIFEST_SIZE bytes)"; fi
    - ANNOTATION_SIZE=$(kubectl annotate --list -f /tmp/manifest-check.yaml 2>/dev/null | grep last-applied | wc -c)
    - if [ "$ANNOTATION_SIZE" -gt 51200 ]; then echo "WARNING: Annotation exceeds 50KB"; exit 1; fi
```

#### 2. Explicit Resource Tracking Method

Configured ArgoCD to use `annotation+labels` explicitly, ensuring consistent tracking:

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  application.resourceTrackingMethod: annotation+labels
```

This ensures ArgoCD uses both labels AND annotations for resource tracking, providing redundancy if one mechanism fails.

#### 3. SSA Handling via Resource Customizations

Configured `resource.customizations` in argocd-cm to handle SSA field conflicts gracefully:

```yaml
data:
  resource.customizations: |
    admissionregistration.k8s.io/MutatingWebhookConfiguration:
      ignoreDifferences: |
        jsonPointers:
        - /metadata/managedFields
    admissionregistration.k8s.io/ValidatingWebhookConfiguration:
      ignoreDifferences: |
        jsonPointers:
        - /metadata/managedFields
    # Ignore managedFields diffs for all resources
    "*":
      ignoreDifferences: |
        jsonPointers:
        - /metadata/managedFields
```

This tells ArgoCD to ignore `managedFields` when computing diffs, preventing SSA conflicts from causing `OutOfSync` status.

#### 4. Reconciliation Timeout

Increased the reconciliation timeout to prevent stale cache scenarios:

```yaml
data:
  timeout.reconciliation: 180s
```

This gives ArgoCD more time to complete reconciliation before marking an application as failed.

#### 5. Monitoring Alerts

Added Prometheus alerting for ArgoCD application health:

```yaml
# PrometheusRule
groups:
- name: argocd-alerts
  rules:
  - alert: ArgoCDAppOutOfSync
    expr: argocd_app_info{sync_status="OutOfSync"} > 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "ArgoCD application {{ $labels.name }} is OutOfSync"

  - alert: ArgoCDAppDegraded
    expr: argocd_app_info{health_status="Degraded"} > 0
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "ArgoCD application {{ $labels.name }} is Degraded"

  - alert: ArgoCDAppComparisonFailed
    expr: argocd_app_info{sync_status="OutOfSync"}
      and on(name) argocd_app_condition{type="ComparisonError"} > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "ArgoCD application {{ $labels.name }} comparison failed"
```

The alert `ArgoCDAppComparisonFailed` would have caught this incident within 1 minute instead of the 5 minutes it took for the platform team to notice.

#### 6. Regular Repo Server Cache Cleanup

Added a cron job to periodically restart the Repo Server during low-traffic periods:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: argocd-repo-server-cache-cleanup
  namespace: argocd
spec:
  schedule: "0 4 * * 0"  # Every Sunday at 4 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: restart
            image: bitnami/kubectl
            command:
            - kubectl
            - rollout
            - restart
            - deployment/argocd-repo-server
            - -n
            - argocd
          restartPolicy: OnFailure
          serviceAccountName: argocd-repo-server
```

#### 7. SSA Field Manager Strategy Documentation

We documented a company-wide strategy for SSA field manager usage across all GitOps tools:

- **ArgoCD**: Uses `argocd-controller` as field manager with `Apply` operation
- **Helm**: Uses `helm` as field manager — must NOT use `--force` on resources managed by ArgoCD
- **Flux**: Uses `flux` as field manager — coordinate with ArgoCD on shared resources
- **Manual kubectl**: Uses `kubectl` as field manager — use `--server-side --force-conflicts` only for emergency overrides

Resources managed by ArgoCD should only be modified by ArgoCD's field manager to avoid conflicts.

## 经验教训 / Lessons Learned

### What Went Well

- **Git history preserved**: All changes were in git, so no data was lost during the recovery
- **Unified tooling**: Using ArgoCD as the single source of truth made it clear which resources were affected
- **kubectl annotation removal**: The `-` suffix syntax (`annotation-`) for removing annotations was quick and effective

### What Could Have Been Better

- **No monitoring on ArgoCD health**: We had monitoring for cluster nodes and pods, but not for ArgoCD application sync status. Basic alerts would have cut detection time from 5 minutes to under 1 minute.
- **No annotation size limits**: We had no guardrails preventing annotations from growing unbounded. A 283KB annotation should never have been allowed to accumulate.
- **Mixed GitOps tools on shared resources**: Helm and ArgoCD both modified the same ConfigMap without coordination. SSA field manager conflicts were a known risk that we failed to address.
- **Cache invalidation strategy**: Relying solely on automatic cache invalidation without understanding the failure modes left us blind when the Repo Server cache became poisoned.
- **No runbook for comparison failed**: When the `comparison failed` error appeared, the team spent 10 minutes researching the error before acting. A documented runbook would have accelerated recovery.

### Key Takeaways

1. **Annotations have real size limits**: The `last-applied-configuration` annotation is not free. In ArgoCD environments, it is read on EVERY reconciliation cycle. Large annotations cause real performance and stability problems.

2. **Server-side apply needs coordination**: SSA is powerful but requires clear ownership. When multiple tools manage the same resource, field manager conflicts are inevitable unless explicitly coordinated.

3. **The SharedInformer is a single point of failure**: A single malformed resource object can corrupt the informer cache and affect ALL applications watching the same cluster. This is an architectural risk to be aware of.

4. **Hard refresh is your friend**: When ArgoCD behaves unexpectedly, `--hard-refresh` forces a complete state reset — re-clone repos, re-generate manifests, re-fetch live state. It is the first troubleshooting step for unexplained diff errors.

5. **Cache poisoning is hard to detect**: A stale or corrupted cache produces errors that look like real application problems. Always consider the cache as a potential root cause when errors seem widespread or inconsistent.

## 总结 / Summary

### Incident Timeline

| Time (UTC) | Event |
|------------|-------|
| 14:30 | Developer pushes ConfigMap update to GitLab |
| 14:32 | CI pipeline completes successfully |
| 14:35 | All 50+ ArgoCD applications show OutOfSync + comparison failed |
| 14:40 | Platform team initiates emergency response |
| 14:45 | Root cause identified: oversized last-applied-configuration annotation (283KB) |
| 14:48 | Corrupted annotation removed via `kubectl annotate ...-` |
| 14:52 | Hard refresh initiated on all applications |
| 15:00 | SSA field manager conflicts resolved with `--force-conflicts` |
| 15:05 | Repo Server restarted to clear cache |
| 15:10 | Bulk sync started |
| 15:20 | All applications back to Synced/Healthy |
| 16:00 | Post-mortem meeting; preventive measures drafted |

### Troubleshooting Command Reference

| Command | Purpose |
|---------|---------|
| `argocd app list` | Quick overview of all application sync/health status |
| `argocd app get <app>` | Detailed status including error conditions |
| `argocd app get <app> --hard-refresh` | Force full state refresh (re-clone, re-generate, re-fetch) |
| `argocd app diff <app>` | Compare desired vs live state for a specific app |
| `argocd app sync <app> --prune` | Trigger sync with resource pruning |
| `argocd repo list` | Verify git repository connectivity |
| `kubectl annotate <res> <name> -n <ns> annotation-` | Remove a problematic annotation (trailing `-`) |
| `kubectl apply --server-side --force-conflicts --field-manager=X -f file` | Force-apply with SSA, resolving field manager conflicts |
| `kubectl rollout restart deploy/argocd-repo-server -n argocd` | Clear Repo Server manifest cache |
| `kubectl logs deploy/argocd-application-controller -n argocd` | Check controller logs for diff/sync errors |
| `kubectl logs deploy/argocd-repo-server -n argocd` | Check repo server logs for manifest generation errors |

### Final Thoughts

This incident was a reminder that GitOps tools like ArgoCD, while powerful, are not immune to failure modes that compound across an entire platform. A 283KB annotation in a single ConfigMap brought down 50+ applications because of how ArgoCD's SharedInformer, diff engine, and cache architecture interact.

The fix was simple — remove the annotation, hard refresh, restart the cache — but the investigation required understanding ArgoCD's internals: how the three-way diff works, how SSA field managers interact, and how the SharedInformer cache can become a single point of failure.

For teams running ArgoCD at scale, the lessons are clear: monitor your annotation sizes, coordinate SSA field manager usage across tools, set up ArgoCD-specific alerts, and always keep `--hard-refresh` in your troubleshooting toolkit.

---

*This article reflects a real production incident. Names and configurations have been generalized to focus on the technical patterns rather than specific details.*
