---
title: "Alertmanager Duplicate Alerts — When PagerDuty Wouldn't Stop Screaming at 3 AM"
date: 2026-06-09
weight: 100460
slug: "alertmanager-duplicate-alerts-silences"
tags: ["prometheus", "alertmanager", "monitoring", "troubleshooting", "alerting"]
categories: ["Troubleshooting"]
description: "An Alertmanager incident — how misconfigured group_by and group_wait settings caused 2,000 duplicate alerts per minute, and how expired silence IDs from a configuration reload broke alert suppression"
keywords: "alertmanager duplicate alerts, prometheus alertmanager silences not working, alertmanager group_by, alertmanager grouping, pagerduty alert storm"
draft: false
featured: true
cover:
  image: ""
  caption: "Alertmanager — Troubleshooting"
---

# Alertmanager Duplicate Alerts — When PagerDuty Wouldn't Stop Screaming at 3 AM

## Common Search Queries

- alertmanager duplicate alerts fix
- prometheus alertmanager silences not working
- alertmanager group_by misconfiguration
- alertmanager silence expired after reload
- alertmanager cluster split brain notifications
- pagerduty alert storm alertmanager
- amtool silence list not matching
- alertmanager group_wait every alert separate
- prometheus alertmanager duplicate pagerduty alerts
- alertmanager cluster gossip failed peers

---

## The Incident

### Environment

| Component | Version / Spec |
|-----------|---------------|
| Prometheus | 2.45 |
| Alertmanager | 0.27 |
| Notification Channels | PagerDuty (primary), Slack (secondary) |
| Cluster Size | 50 Kubernetes nodes, 3 Alertmanager replicas |
| Alerting Rules | 2,000+ recording and alerting rules |

### Timeline

**02:45 AM** — A rolling update of a Kubernetes `DaemonSet` across 50 nodes was triggered. Each node running the DaemonSet restarted its Pod, which caused a cascade of `KubePodCrashLooping`, `PodNotReady`, and `NodeCondition` alerts. The on-call engineer had intentionally created silences earlier that evening for planned maintenance on a different component — those silences should have suppressed some of the noise.

**02:47 AM** — PagerDuty began firing. The engineer's phone lit up with 50+ notifications per minute. The Slack `#alerts` channel was flooded at a rate of 30 messages per minute. The PagerDuty mobile app crashed twice due to notification overload.

**02:52 AM** — The engineer acknowledged the PagerDuty incident and checked the Alertmanager status UI, expecting to see grouped alerts. Instead they found 200+ distinct alert groups, each with a single alert.

**02:55 AM** — The engineer attempted to silence the flooding alerts using `amtool silence add`, only to discover that existing silences — created just hours earlier — were no longer suppressing alerts. The `amtool silence list` showed the silences as active, but they had zero effect.

**03:00 AM** — After 15 minutes of chaos, the engineer had received over 200 PagerDuty push notifications and 500+ Slack messages. The production incident was escalated to SRE lead.

### Symptoms

- **PagerDuty alert storm**: 50+ notifications per minute, mobile app crashed twice
- **Silence failure**: Previously working maintenance silences had no effect
- **No grouping**: Every Pod restart triggered a separate, independent alert notification
- **Slack flood**: `#alerts` channel had 500+ messages in 15 minutes
- **On-call burnout**: Engineer received 200+ missed calls and 500+ Slack alerts

---

## Background

To understand what went wrong, we need to cover how Alertmanager processes alerts.

### Alertmanager Architecture

Alertmanager is the routing and notification layer of the Prometheus monitoring stack. It receives alerts from Prometheus, then processes them through three stages:

1. **Inhibition** — Suppresses certain alerts when other alerts are firing (e.g., suppress `PodCrashLooping` when `NodeDown` is active).
2. **Grouping** — Collapses similar alerts into a single notification based on label matching.
3. **Silencing** — Suppresses notifications for alerts matching user-defined label matchers.

### Configuration vs Runtime State

A critical distinction in Alertmanager is the difference between configuration and runtime state:

- **Configuration** (`alertmanager.yml`): Defines routes, receivers, grouping rules, inhibition rules, and notification templates. This is loaded from disk at startup or on reload.
- **Runtime State**: Active silences, notification histories, and cluster membership. Silences created via `amtool` or the API exist **only in runtime state** — they are **not** part of `alertmanager.yml`.

When Alertmanager reloads its configuration (via `kill -HUP` or `POST /-/reload`), it re-reads `alertmanager.yml` from disk but **preserves** in-memory silences — **unless** the reload also specifies a new silence file path or the silence data directory is lost. This is where many teams get tripped up.

### Cluster Mode and Gossip Protocol

Alertmanager supports high-availability mode by running multiple replicas. Replicas communicate via a **gossip protocol** (based on Protobuf over TCP, port 9094 by default) to:

- Synchronize active silences across all instances
- Deduplicate notification delivery — only one replica should send a notification per alert group
- Share notification history to prevent repeated sends after leader changes

Gossip requires stable network connectivity between all peers. If a peer is unreachable, it becomes split-brain, and each partition will independently fire notifications.

---

## Investigation

The on-call engineer and the SRE lead followed this systematic investigation process:

### Step 1: Check Alertmanager Status UI

```bash
# Query the Alertmanager status endpoint
curl -s http://localhost:9093/api/v2/status | jq .
```

The response revealed:

```json
{
  "config": { ... },
  "cluster": {
    "status": "ready",
    "peers": [
      {"name": "pod-0", "address": "10.0.1.10:9094"},
      {"name": "pod-1", "address": "10.0.1.11:9094"},
      {"name": "pod-2", "address": "10.0.1.12:9094"}
    ]
  },
  "uptime": "2026-06-08T14:00:00Z",
  "versionInfo": {"version": "0.27.0"}
}
```

All three peers were visible, but the cluster gossip stats showed failures.

### Step 2: Check Active Alerts

```bash
amtool alert list
```

Output showed 1,200+ active alerts — each with unique label combinations. This was expected during the rolling update, but the grouping should have collapsed them.

### Step 3: Check Silences

```bash
amtool silence list
```

Silences were present and showed `status: active`:

```
ID         Matchers                         Status   Ends
abc123     alertname="NodeNotReady"         Active   2026-06-09 06:00:00
def456     alertname=".*" pod="web-.*"      Active   2026-06-09 06:00:00
```

Yet alerts matching these patterns were still firing notifications. This was the first red flag — silences were **active but not matching**.

### Step 4: Compare Silence Matchers with Alert Labels

```bash
# Check labels on a specific alert
amtool alert query alertname="NodeNotReady" --json | jq '.[0].labels'
```

```json
{
  "alertname": "NodeNotReady",
  "instance": "node-23",
  "namespace": "production",
  "node_name": "node-23",
  "pod": "",
  "severity": "critical"
}
```

Now compare with the silence matcher:

```
Silence: alertname="NodeNotReady"
```

This should match. But the SRE lead noticed a subtlety — the second silence used `pod="web-.*"` but the alerts had a different label name. After the last configuration change, the label `pod` was renamed to `pod_name` in the alerting rules, but the silence still matched on the old `pod` label.

Additionally, some silences had been created with `regex: true` but the regex pattern was syntactically incorrect after the label change, causing a no-match.

### Step 5: Check Grouping Configuration

```bash
amtool config show
```

The `route` section revealed the root cause of the duplicate notifications:

```yaml
route:
  receiver: "pagerduty-critical"
  group_by: ['...']
  group_wait: 5s
  group_interval: 5m
  repeat_interval: 4h
```

**`group_by: ['...']`** is a trap. The single `...` (ellipsis/three dots) instructs Alertmanager to group by **all** labels. This means every unique combination of label values creates its own alert group. With 50 nodes and 10 different alert types, this produced 500+ distinct groups, each triggering its own notification.

The `group_wait: 5s` made things worse — it gave only 5 seconds for alerts to arrive before firing a notification. With a rolling update, new alerts were being generated continuously, but 5 seconds was far too short to batch them effectively.

### Step 6: Check Alertmanager Logs

```bash
kubectl logs -n monitoring alertmanager-0 | grep -i error
```

Logs showed:

```
level=error msg="silence abc123: failed to match" error="regex compile error"
level=warn msg="cluster gossip: 2 of 3 peers unreachable" count=2
level=error msg="notification failed: rate limit exceeded" receiver=pagerduty
```

Three issues confirmed:
1. Silence regex compilation failure
2. Cluster gossip problems — split-brain
3. PagerDuty rate limiting due to the notification storm

### Step 7: Check Gossip / Cluster Status

```bash
curl -s http://localhost:9093/api/v2/status | jq '.cluster'
```

```json
{
  "status": "ready",
  "peers": [
    {"name": "pod-0", "address": "10.0.1.10:9094"},
    {"name": "pod-1", "address": "10.0.1.11:9094"},
    {"name": "pod-2", "address": "10.0.1.12:9094"}
  ],
  "failedPeers": [
    {"name": "pod-1", "address": "10.0.1.11:9094"},
    {"name": "pod-2", "address": "10.0.1.12:9094"}
  ]
}
```

Pod-0 was isolated — pods 1 and 2 were unreachable on the gossip port. Network policy audit revealed that a recent `NetworkPolicy` update had blocked port 9094 between certain pods.

With 2 out of 3 peers unreachable, the Alertmanager cluster was effectively split-brain. Pod-0 was firing its own notifications, pod-1 and pod-2 were each firing their own — tripling the notification volume.

### Step 8: Check Notification Metrics

```bash
# PromQL query
rate(alertmanager_notifications_total{receiver="pagerduty"}[5m])
```

The query returned **2,000 notifications per minute** — far exceeding the PagerDuty rate limit of 120 notifications per minute. PagerDuty's rate limiting then caused back-pressure, leading to failed notifications and retries, which generated even more noise.

---

## Root Cause

Five compounding factors created this incident:

### 1. `group_by: ['...']` — Global Grouping

**`group_by: ['...']`** (three dots) is equivalent to grouping by **every** label. It is almost never the right choice in production:

- Each unique combination of label values creates a separate alert group
- A rolling update across 50 nodes with 10 alert types = 500+ groups
- Each group fires its own notification to PagerDuty

The correct configuration is to specify explicit, cardinality-limiting labels:

```yaml
# Bad — creates a group per unique label combination
group_by: ['...']

# Good — groups by alert name and severity
group_by: ['alertname', 'severity']
```

### 2. `group_wait: 5s` — Insufficient Batching Window

`group_wait` controls how long Alertmanager waits for additional alerts before sending the first notification for a group. With only 5 seconds:

- Alerts for the same node that arrive 6 seconds apart fire as separate notifications
- During a rolling update, batches of alerts arrive over minutes, not seconds
- A 30-second to 1-minute `group_wait` would have collapsed many alerts into a single notification

### 3. Silence State Lost After Config Reload

This was the most insidious issue. Earlier that evening, the team had applied a configuration change to Alertmanager:

```bash
kill -HUP $(pidof alertmanager)
# or
curl -X POST http://localhost:9093/-/reload
```

Normally, Alertmanager reloads preserve in-memory silences. But in this case, the reload occurred with a **modified startup command** that pointed to a new data directory (`--data.dir`), causing Alertmanager to initialize with an empty silence store. All active silences created via the API and `amtool` were **orphaned** — they still appeared in `amtool silence list` (from the old store) but existed as stale references that no longer matched.

The silences showed `status: active` but their internal state had been corrupted. Alertmanager could not properly evaluate them against incoming alerts.

### 4. Silence Regex Mismatch After Label Rename

During a previous Prometheus relabeling change, `pod` was renamed to `pod_name` in alert labels. The silence matcher still referenced the old `pod` label:

```
# Old silence — no longer matches
pod="web-.*"  (regex: true)

# New label on alerts after relabeling change
pod_name="web-frontend-abc123"
```

Alertmanager's silence matching is strict — if the label doesn't exist on the alert, the matcher simply doesn't fire. The silence appeared active but was effectively useless.

### 5. Cluster Split-Brain — Gossip Failure

A `NetworkPolicy` update blocked TCP port 9094 between Alertmanager pods, breaking gossip synchronization:

| Pod | Status | Impact |
|-----|--------|--------|
| pod-0 | Isolated, can't reach pod-1 or pod-2 | Fires own notifications |
| pod-1 | Can't reach pod-0, can reach pod-2 | Fires own notifications, duplicate with pod-0 |
| pod-2 | Can't reach pod-0, can reach pod-1 | Fires own notifications, duplicate with pod-0 |

Instead of one notification per alert group, the cluster sent **three** — one from each partition. This tripled the notification volume and contributed to the PagerDuty rate limit breach.

---

## Resolution

### Emergency Response

**1. Expire broken silences and recreate them:**

```bash
# Clear all broken silences
amtool silence expire --all

# Recreate silences with correct matchers
amtool silence add \
  alertname="NodeNotReady" \
  matcher="severity=critical" \
  --duration=4h \
  --comment="Maintenance window - rolling update"

# Add silence with correct label name (pod_name, not pod)
amtool silence add \
  alertname=".*" \
  matcher="pod_name=web-.*" \
  matcher="severity=critical" \
  --duration=4h \
  --comment="Web service rolling update"

# Verify silence is matching
amtool silence add --verify \
  alertname="KubePodCrashLooping" \
  matcher="severity=warning" \
  --duration=1h
```

**2. Fix grouping configuration:**

```bash
# Edit alertmanager.yml
---
route:
  receiver: "pagerduty-critical"
  group_by: ['alertname', 'namespace', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**3. Reload with consistent data directory:**

```bash
# Ensure data directory is preserved
alertmanager --config.file=/etc/alertmanager/alertmanager.yml \
  --data.dir=/var/lib/alertmanager \
  --cluster.listen-address=0.0.0.0:9094

# Reload configuration
kill -HUP $(pidof alertmanager)
```

**4. Fix cluster networking:**

```bash
# Update NetworkPolicy to allow gossip traffic on port 9094
# After fix, verify cluster health
curl -s http://localhost:9093/api/v2/status | jq '.cluster.failedPeers'
# Expected: [] (empty array)
```

**5. Pause PagerDuty auto-acknowledgment:**

As a last resort, the PagerDuty integration was temporarily disabled for 10 minutes to stop the notification flood, then re-enabled after the fix was applied.

### Verified Fix

After applying the changes:

```bash
# Verify no failed peers
curl -s http://localhost:9093/api/v2/status | jq '.cluster.failedPeers'
# → []

# Verify notifications dropped
# PromQL: rate(alertmanager_notifications_total[5m])
# Before: 2,000/min
# After: ~5/min (grouped into meaningful batches)

# Verify silences match
amtool silence list
# Shows new silences actively suppressing
```

### Long-Term Remediation

**1. Infrastructure-as-Code for Silences:**

Store Alertmanager silences in Terraform or Helm values:

```terraform
resource "alertmanager_silence" "maintenance" {
  matchers {
    name    = "alertname"
    value   = "NodeNotReady"
    is_regex = false
  }
  matchers {
    name    = "severity"
    value   = "critical"
    is_regex = false
  }
  duration = "4h"
  comment  = "Maintenance window - rolling update"
}
```

Or for Helm-deployed Alertmanager, use `extraSilences` in values:

```yaml
# values.yaml
alertmanager:
  extraSilences:
    - matchers:
        - name: alertname
          value: "NodeNotReady"
      duration: 4h
      comment: "Maintenance window"
```

**2. Explicit Grouping Labels — Never Use `group_by: ['...']`:**

```yaml
# Always specify explicit grouping labels
route:
  receiver: "pagerduty-critical"
  group_by: ['alertname', 'namespace', 'severity', 'job']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
```

**3. Monitor Cluster Health:**

Add these alerts to Prometheus:

```yaml
groups:
  - name: alertmanager-cluster
    rules:
      - alert: AlertmanagerClusterSplitBrain
        expr: |
          count(alertmanager_cluster_members) > 1
          and
          (count(alertmanager_cluster_members)
           - count(alertmanager_cluster_failed_peers) == 1)
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Alertmanager cluster may be split-brain"

      - alert: AlertmanagerClusterFailedPeers
        expr: |
          alertmanager_cluster_failed_peers > 0
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Alertmanager cluster has failed peers"
```

**4. Test Silence Matchers Before Applying:**

```bash
# Verify silence matches before creating it
amtool silence add --verify \
  alertname="KubePodCrashLooping" \
  matcher="namespace=production" \
  --duration=1h

# Or test with an existing alert
amtool silence add --dry-run \
  alertname="KubePodCrashLooping" \
  matcher="namespace=production" \
  --duration=1h
```

**5. Pin Configuration in Git + CI/CD:**

- Store `alertmanager.yml` in version control
- Use CI/CD pipelines for config changes (ArgoCD, Flux, or custom scripts)
- Add automated validation: `amtool check-config alertmanager.yml`
- Never `kill -HUP` from a shell without verifying the config is correct

**6. PagerDuty Alert Fatigue Rule:**

Create a PagerDuty "Alert Fatigue" rule that throttles notifications from a single Alertmanager receiver to a maximum rate (e.g., 30 notifications per 5 minutes) with a warning to the operations team instead of silencing critical alerts.

---

## Lessons Learned

### What Went Well

- The on-call engineer had Alertmanager's `amtool` CLI ready and knew the key diagnostic commands
- The SRE lead joined quickly and the team had a structured investigation process
- The Alertmanager status API (`/api/v2/status`) provided immediate visibility into cluster health
- Prometheus metrics (`alertmanager_notifications_total`) gave quantitative confirmation of the problem and the fix

### What Went Wrong

| Issue | Category | Impact |
|-------|----------|--------|
| `group_by: ['...']` | Configuration | 500+ separate alert groups |
| `group_wait: 5s` | Configuration | No batching of alerts |
| Silence state lost after reload | Operational | Suppression broken for known issues |
| Silence label mismatch | Configuration | Regex matchers failed silently |
| Cluster gossip blocked | Network | 3x notification duplication |
| No cluster health monitoring | Observability | Split-brain undetected for hours |

### Key Takeaways

1. **Never use `group_by: ['...']` in production** — always specify explicit, cardinality-limiting labels. Think carefully about what labels produce useful alert groups (e.g., `alertname` + `severity` + `namespace`).

2. **Alertmanager silences are runtime state** — they live in memory, not in the config file. Back them up, store them as code, and never assume a config reload is harmless.

3. **Test silence matchers rigorously** — use `amtool silence add --verify` to confirm matchers match real alerts. A silence that doesn't match is worse than no silence at all (it creates a false sense of security).

4. **Monitor the monitoring system** — Alertmanager cluster health, gossip status, and notification rates should be part of your standard observability stack. If your monitoring system breaks, you're flying blind.

5. **The `--data.dir` matters on reload** — when reloading Alertmanager, ensure the data directory is consistent. A reload with a different `--data.dir` is effectively a restart with an empty silence store.

6. **NetworkPolicy audit for control plane** — the Kubernetes control plane components need stable network connectivity. A `NetworkPolicy` that inadvertently blocks gossip ports can cause split-brain across any stateful distributed system.

---

## Summary

### Timeline Recap

| Time | Event |
|------|-------|
| 02:45 | Rolling update triggered across 50 nodes |
| 02:47 | PagerDuty storm begins — 50+ notifications/min |
| 02:50 | Slack `#alerts` channel flooded |
| 02:52 | Engineer checks Alertmanager UI — 200+ distinct alert groups |
| 02:55 | Engineer discovers silences are not matching |
| 03:00 | 200+ PagerDuty notifications, 500+ Slack messages |
| 03:05 | SRE lead joins — begins structured investigation |
| 03:20 | Root cause identified |
| 03:25 | Emergency fix deployed: silences recreated, grouping fixed, network policy updated |
| 03:30 | Notifications drop to ~5/min |
| 03:45 | Incident declared resolved |

### Before vs After Configuration

```yaml
# BEFORE — BROKEN
route:
  group_by: ['...']
  group_wait: 5s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
```

```yaml
# AFTER — FIXED
route:
  group_by: ['alertname', 'namespace', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match:
        severity: critical
      receiver: pagerduty-critical
    - match:
        severity: warning
      receiver: slack-warnings
```

### Key Metrics

| Metric | Before | After |
|--------|--------|-------|
| Notifications per minute | 2,000 | ~5 |
| Distinct alert groups | 500+ | 5-10 |
| Cluster failed peers | 2 | 0 |
| Silence match rate | ~0% (broken) | 100% |
| PagerDuty rate limit hits | Yes | No |
| Engineer pager fatigue | Critical | Normal |

### Final Verdict

This incident was not caused by a single failure but by a **cascade of configuration errors** that compounded each other: wrong grouping strategy, insufficient batch window, lost silence state, label drift between silences and alerts, and a network policy that fragmented the cluster. Each issue alone would have been manageable. Together, they created one of the most disruptive on-call experiences imaginable.

The fixes were straightforward — explicit `group_by` labels, longer `group_wait`, Infrastructure-as-Code for silences, and cluster health monitoring — but they required a painful incident to be prioritized. As with many SRE lessons, the cost of learning the hard way was a 3 AM PagerDuty storm that would not stop screaming.

---

*First-hand account from the SRE team. Names and some details have been generalized, but the technical root causes are accurate as documented.*
