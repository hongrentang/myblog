---
title: "Audit Log Missing — When Your Kubernetes Cluster Was Breached and You Have No Forensics"
date: 2026-06-03
weight: 100320
slug: "audit-log-missing-forensics"
tags: ["kubernetes", "security", "audit", "forensics", "troubleshooting"]
categories: ["Security"]
description: "A security incident investigation crippled by missing Kubernetes audit logs — how a misconfigured audit policy and lack of log persistence left the security team blind after a cluster breach"
keywords: "kubernetes audit log missing, k8s forensics failure, audit policy misconfiguration, cluster breach investigation, kubernetes security incident response"
draft: false
featured: true
cover:
  image: ""
  caption: "Audit Log Missing — Forensics Investigation"
---

# Audit Log Missing — When Your Kubernetes Cluster Was Breached and You Have No Forensics

## Common Search Queries

- kubernetes audit log not working
- k8s audit policy misconfiguration
- kubernetes forensics no audit logs
- audit log level metadata vs requestresponse
- kubernetes audit log shipping fluentd
- kube-apiserver audit logging best practices
- kubernetes incident response no logs

---

## The Incident

**Environment**: K8S v1.28, kubeadm deployed, 3 control plane nodes (HA), 50 worker nodes, on-premise datacenter. Audit log storage configured on local control plane disk with no external shipping.

**Time**: Saturday 03:14. PagerDuty alert: `NodePortExhaustion` on 12 worker nodes simultaneously. Seconds later, `CPUThrottlingHigh` on the `kube-system` namespace. Then `KubeAPILatencyHigh` — the API Server response time spiked from 50ms to 12s.

**Initial Discovery**: The on-call engineer logged into the control plane and found a cryptominer pod running in the `kube-system` namespace, disguised as `kube-proxy-monitor`:

```bash
# What the engineer saw
kubectl get pods -n kube-system | grep -v Running
NAME                               READY   STATUS    RESTARTS   AGE
kube-proxy-monitor                 1/1     Running   0          2h
```

```bash
# Describe revealed a suspicious image
kubectl describe pod kube-proxy-monitor -n kube-system | grep Image
Image:          docker.io/malregistry/kube-proxy-monitor:v1.0.0
```

This image was not from the organization's internal registry. The pod had been created 2 hours ago, and it was running a cryptominer that was consuming 8 CPU cores and 6 GB of RAM on the control plane node.

**Impact**:
- Cluster performance degraded by 60% across all nodes
- 12 worker nodes hit port exhaustion — critical ingress traffic dropped
- API Server latency spiked, causing controller manager leader election failures
- The cryptominer ran for ~2 hours before detection
- **Zero forensic evidence for how the attacker got in**

---

## Background

The cluster was deployed 14 months ago using kubeadm. At the time, audit logging was an afterthought. The team had enabled it with a minimal policy because "the cluster is behind the firewall" and "we have network policies."

### The Original Audit Policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata   # ← Only logs metadata, NOT request/response bodies
  resources:
  - group: ""        # core group
    resources: ["pods", "services", "configmaps", "secrets"]
- level: None        # ← Ignores all other resources
  resources:
  - group: ""
    resources: ["events", "endpoints"]
- level: None
  users: ["system:kube-controller-manager", "system:kube-scheduler"]
```

**Key misconfigurations in this policy:**
1. `level: Metadata` — logs only the user, timestamp, and resource, but **not** the request body or response body. For forensics, this means you can see *that* a pod was created, but not *what* the pod spec contained
2. `level: None` for controller manager and scheduler — suppresses all their actions, including any anomalous API calls made by a compromised component
3. No rule for `ClusterRole`/`ClusterRoleBinding` resources — RBAC tampering is invisible
4. No catch-all rule at the bottom — any resource type not explicitly listed is **not audited**

### The Log Storage Config

```bash
# kube-apiserver manifest snippet
spec:
  containers:
  - command:
    - kube-apiserver
    - --audit-log-path=/var/log/kubernetes/audit.log
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-maxage=7       # ← Delete logs after 7 days
    - --audit-log-maxbackup=5     # ← Keep only 5 rotated files
    - --audit-log-maxsize=100     # ← Rotate at 100 MB each
```

The logs were stored **only** on the control plane node's local disk. There was no log shipping to a centralized SIEM. The `--audit-log-maxage=7` setting meant logs older than 7 days were auto-deleted.

### The Problem

When the breach was discovered at 03:14, the security team went looking for audit logs. Here is what they found:

```bash
# Check if audit logging is enabled
ls -la /var/log/kubernetes/audit.log
-rw------- 1 root root 0 Jun 3 03:20 /var/log/kubernetes/audit.log
# File was empty — it had been rotated and the new file started at 03:20
# because the API server was restarted during incident containment
```

```bash
# Check rotated logs
ls -la /var/log/kubernetes/audit.log.*
audit.log.20260602   # ← Only 1 rotated file, from yesterday
audit.log.20260601   # ← And one more from the day before
```

Only 2 days of rotated logs existed. The breach happened 2 hours ago, but the audit logs from the exact time window were **missing** because the API Server was restarted during the initial response.

---

## Investigation

### Step 1: Confirm Audit Logging Status

```bash
# On the control plane node
kubectl -n kube-system get pods -l component=kube-apiserver -o jsonpath='{.items[0].spec.containers[0].command}' | jq .
```

```json
[
  "kube-apiserver",
  "--audit-log-path=/var/log/kubernetes/audit.log",
  "--audit-policy-file=/etc/kubernetes/audit-policy.yaml",
  "--audit-log-maxage=7",
  "--audit-log-maxbackup=5",
  "--audit-log-maxsize=100"
]
```

Audit logging was "enabled" — but with severe limitations.

### Step 2: Examine the Audit Policy

```bash
cat /etc/kubernetes/audit-policy.yaml
```

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["pods", "services", "configmaps", "secrets"]
- level: None
  resources:
  - group: ""
    resources: ["events", "endpoints"]
- level: None
  users: ["system:kube-controller-manager", "system:kube-scheduler"]
```

The security team immediately identified the gaps:
- `level: Metadata` only captures `User`, `Timestamp`, `Verb`, `Object`, `SourceIPs` — no request body
- RBAC changes (`ClusterRole`/`ClusterRoleBinding`) are **not in the rule list** — they fall through to the default, which is... nothing

```bash
# What is the default audit level?
# Kubernetes documentation: if no matching rule is found, the request is NOT logged
```

That was the critical gap. The attacker could have created a ClusterRoleBinding, and there would be **no audit record** of it.

### Step 3: Check What Logs Exist

```bash
# Parse the existing audit logs for the breach timeframe
grep "03:00" /var/log/kubernetes/audit.log.* 2>/dev/null | head -5
# No output — logs from 03:00-03:20 were lost when API server restarted
```

```bash
# Check what we DO have
cat /var/log/kubernetes/audit.log.20260602 | wc -l
81234

cat /var/log/kubernetes/audit.log.20260601 | wc -l
120456
```

The logs from yesterday and the day before existed, but **the 2-hour breach window was gone**.

### Step 4: Try to Reconstruct the Attack Vector

```bash
# Look for ClusterRoleBinding changes in available logs
grep -i "clusterrolebinding" /var/log/kubernetes/audit.log.* | head -10
# No matches — ClusterRoleBinding operations were never audited
```

```bash
# Look for suspicious kubectl exec or pod creation in available logs
grep "create.*pods" /var/log/kubernetes/audit.log.20260602 | head -20
```

```
2026-06-02T22:15:00Z  admin@company.com  create  pods  kube-system  deployment=...
2026-06-02T22:30:00Z  admin@company.com  create  pods  default  deployment=...
2026-06-02T23:45:00Z  system:node:cp-1  create  pods  kube-system  kube-proxy-monitor
```

Wait — there it was. `system:node:cp-1` created the `kube-proxy-monitor` pod at 23:45, **3 hours before the alert**. The identity `system:node:cp-1` means the API call was made using the kubelet credentials of the control plane node itself.

```bash
# Investigate the kubelet on cp-1
journalctl -u kubelet -n 200 --no-pager | grep -i "kube-proxy-monitor"
```

```
Jun 2 23:44:58 cp-1 kubelet[2345]: E2345 api.go:123] Failed to create pod kube-proxy-monitor: node "cp-1" not found
Jun 2 23:44:59 cp-1 kubelet[2345]: I2345 kubelet.go:201] SyncLoop (PLEG): event for pod kube-proxy-monitor
Jun 2 23:45:01 cp-1 kubelet[2345]: I2345 kubelet.go:201] SyncLoop (PLEG): event for pod kube-proxy-monitor
```

The kubelet logs confirmed the pod was being managed by the kubelet, but they didn't explain **who told the kubelet to create it**, because the kubelet doesn't do that directly.

```bash
# Check the kubelet configuration for static pod paths
grep -i "staticpodpath\|staticPodPath\|--pod-manifest-path" /var/lib/kubelet/config.yaml
```

```
staticPodPath: /etc/kubernetes/manifests
```

```bash
# Check the static pod directory for any unexpected manifest
ls -la /etc/kubernetes/manifests/
total 28
drwxr-xr-x 2 root root 4096 Jun  2 23:44 .
drwxr-xr-x 4 root root 4096 Jun  2 10:00 ..
-rw------- 1 root root 2002 Jun  2 10:00 kube-apiserver.yaml
-rw------- 1 root root 1856 Jun  2 10:00 kube-controller-manager.yaml
-rw------- 1 root root 1440 Jun  2 10:00 kube-scheduler.yaml
-rw-r--r-- 1 root root  422 Jun  2 23:44 kube-proxy-monitor.yaml   # ← ATTACKER LEFT THIS
```

The attacker had written a **static pod manifest** directly to `/etc/kubernetes/manifests/` on the control plane node. The kubelet watches this directory and automatically creates pods from any manifests placed there. This bypassed the API Server entirely — **no audit record would ever be generated**, because the pod was never created via the API.

### Step 5: Determine How the Attacker Got Shell Access

```bash
# Check SSH auth logs
grep "Accepted\|Failed" /var/log/auth.log | tail -50
```

```
Jun  2 23:30:01 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:05 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:09 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:12 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:15 cp-1 sshd[12345]: Failed password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:18 cp-1 sshd[12345]: Accepted password for root from 10.0.0.99 port 54321 ssh2
Jun  2 23:30:18 cp-1 sshd[12345]: lastlog: control process exited
```

SSH brute force succeeded. The attacker got root access to the control plane node via SSH, then deployed a static pod manifest. But the security team still had questions:

- Was `10.0.0.99` a known internal IP or an external one?
- How did the attacker know the root password?
- Was this a single-node compromise or were other nodes affected?
- Were there lateral movements within the cluster?

The audit logs could not help answer any of these questions.

### Step 6: Check the Other Two Control Plane Nodes

```bash
for node in cp-2 cp-3; do
  echo "=== $node ==="
  ssh $node "ls -la /etc/kubernetes/manifests/; echo '---'; grep 'kube-proxy-monitor' /var/log/kubernetes/audit.log 2>/dev/null"
done
```

```
=== cp-2 ===
total 24
drwxr-xr-x 2 root root 4096 Jun  2 10:00 .
drwxr-xr-x 4 root root 4096 Jun  2 10:00 ..
-rw------- 1 root root 2134 Jun  2 10:00 kube-apiserver.yaml
-rw------- 1 root root 1856 Jun  2 10:00 kube-controller-manager.yaml
-rw------- 1 root root 1440 Jun  2 10:00 kube-scheduler.yaml
(No evidence of attacker on cp-2)

=== cp-3 ===
total 24
(No suspicious files, grep returned nothing)
```

Only cp-1 was compromised. But without centralized audit logs, the team could not determine whether the attacker had made any API calls from cp-1.

### Step 7: Attempt to Extract What Little Audit Data Exists

```bash
# Extract any information about the attacker's API activity
cat /var/log/kubernetes/audit.log.20260602 | jq 'select(.user.username | test("system:node")) | {user: .user.username, verb: .verb, objectRef: .objectRef, sourceIPs: .sourceIPs, requestReceivedTimestamp: .requestReceivedTimestamp}' 2>/dev/null | head -30
```

```json
{
  "user": "system:node:cp-1",
  "verb": "create",
  "objectRef": {
    "resource": "pods",
    "namespace": "kube-system",
    "name": "kube-proxy-monitor"
  },
  "sourceIPs": ["127.0.0.1"],
  "requestReceivedTimestamp": "2026-06-02T23:45:01Z"
}
```

The source IP is `127.0.0.1` — localhost. The API call was made from the local kubelet on cp-1, which is consistent with a static pod manifest. But with `level: Metadata`, the **request body** (what was in the pod spec) was **not logged**. The `responseStatus` was also not logged, so the team couldn't determine if the API call succeeded or failed based on the audit record alone.

---

## Root Cause

1. **Audit logging was not enabled for all resources**: The audit policy only tracked a subset of resources at `Metadata` level. RBAC resources (`ClusterRole`, `ClusterRoleBinding`), `Node` operations, and custom resource definitions were all excluded from audit logging.

2. **Audit log level set to `Metadata` instead of `RequestResponse`**: At `Metadata` level, the log records who, what, and when — but not the request body. This means the attacker's pod spec, commands, and payloads are invisible. For forensic investigations, `RequestResponse` level is essential because it captures the full request and response payload.

3. **Audit logs stored only on local control plane disk**: No log shipping to a centralized SIEM or object storage. When the API Server was restarted during incident response (a common first step), the active audit log file was truncated/rotated, and the logs from the critical 2-hour window were **permanently lost**.

4. **Short log retention period**: `--audit-log-maxage=7` meant logs older than 7 days were deleted. Combined with `--audit-log-maxbackup=5`, the cluster only retained at most 500 MB of audit logs. For a 50-node production cluster generating thousands of API calls per hour, this means logs from just the past few days were available.

5. **No alerting on audit log anomalies**: Even if the audit logs had been complete, there was no system consuming them to detect anomalies like "a kubelet created a pod in kube-system" or "a new ClusterRoleBinding appeared at 3 AM."

6. **The attacker bypassed the API Server entirely**: The attacker gained root SSH access to the control plane node and deployed a static pod manifest directly to `/etc/kubernetes/manifests/`. Static pods are managed by the kubelet directly and **never pass through the API Server**, so they generate **zero audit log entries** regardless of the audit policy configuration. This is a fundamental blind spot in Kubernetes audit logging.

### Audit Policy Level Comparison

| Level | Logged Information | Forensic Value |
|-------|--------------------|----------------|
| `None` | Nothing | None |
| `Metadata` | User, timestamp, verb, object reference, source IP, response status | Low — you know *something* happened, but not *what* |
| `Request` | Metadata + request body | Medium — you can see the request payload |
| `RequestResponse` | Metadata + request body + response body | High — you can see the exact API interaction |

---

## Resolution

### Emergency (Immediate)

```bash
# 1. Remove the malicious static pod
kubectl delete pod kube-proxy-monitor -n kube-system --force --grace-period=0
rm -f /etc/kubernetes/manifests/kube-proxy-monitor.yaml

# 2. Block the attacker IP on the firewall
iptables -A INPUT -s 10.0.0.99 -j DROP

# 3. Force rotate credentials — all service accounts, kubelet certs
kubectl delete secret -n kube-system --all
# Re-deploy all nodes with fresh kubelet certificates

# 4. Audit the SSH access — check which user/key was compromised
grep "10.0.0.99" /var/log/auth.log
```

### Fix the Audit Policy

```yaml
# /etc/kubernetes/audit-policy.yaml — proper configuration
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log all RBAC changes at RequestResponse level
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]

# Log pod operations at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["pods"]

# Log secret and configmap operations at RequestResponse
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

# Log node operations
- level: Request
  resources:
  - group: ""
    resources: ["nodes"]

# Log all other resource operations at Metadata level
- level: Metadata
  resources:
  - group: ""
    resources: ["services", "endpoints", "namespaces", "deployments", "statefulsets", "daemonsets"]

# Log all operations from non-human users at Metadata level (detect machine-driven anomalies)
- level: Metadata
  users: ["system:node:*"]
  userGroups: ["system:nodes"]

# Catch-all — log everything else at Request level
- level: Request
  omitStages:
  - "RequestReceived"
```

### Set Up Audit Log Shipping

```yaml
# fluentd-config.yaml — ship audit logs to S3/Elasticsearch
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-audit-config
  namespace: kube-system
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/kubernetes/audit.log
      pos_file /var/log/fluentd-audit.log.pos
      tag kubernetes.audit
      format json
      read_from_head false
    </source>

    <filter kubernetes.audit>
      @type record_transformer
      <record>
        cluster_name prod-cluster
        hostname "#{Socket.gethostname}"
      </record>
    </filter>

    <match kubernetes.audit>
      @type s3
      s3_bucket prod-k8s-audit-logs
      s3_region us-west-2
      path audit/${hostname}/
      <buffer>
        @type file
        path /var/log/fluentd-buffer
        timekey 3600
        timekey_wait 600
        timekey_use_utc true
      </buffer>
      <format>
        @type json
      </format>
    </match>
```

Add retention at the storage layer:

```bash
# S3 lifecycle policy — retain audit logs for 365 days
aws s3api put-bucket-lifecycle-configuration \
  --bucket prod-k8s-audit-logs \
  --lifecycle-configuration '{
    "Rules": [{
      "Id": "audit-log-retention",
      "Status": "Enabled",
      "Filter": {"Prefix": "audit/"},
      "Expiration": {"Days": 365}
    }]
  }'
```

### Set Up SIEM Alerting

```bash
# Elasticsearch query: detect pods created by nodes (not humans)
# This is highly suspicious — nodes should not create pods via API
POST /k8s-audit-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "user.username": "system:node:" }},
        { "match": { "verb": "create" }},
        { "match": { "objectRef.resource": "pods" }}
      ]
    }
  }
}
```

```yaml
# Prometheus alert for audit log gaps
- alert: AuditLogShippingLag
  expr: time() - fluentd_buffer_tail < 300
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Audit log shipping is lagging by {{ $value }} seconds"

- alert: AuditLogMissing
  expr: absent(rate(apiserver_audit_event_total[5m]))
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "No audit events detected — audit log may be disabled or failed"
```

### Monitor Static Pod Changes

```yaml
# Falco rule: detect new files in the static pod manifest directory
- rule: Static Pod Manifest Change
  desc: Detect new or modified files in /etc/kubernetes/manifests/
  condition: >
    open_write and
    fd.name startswith /etc/kubernetes/manifests/ and
    not proc.name in (kubelet, kube-apiserver, kube-controller-manager, kube-scheduler)
  output: >
    Static pod manifest changed (file=%fd.name process=%proc.name user=%user.name)
  priority: CRITICAL
```

---

## Lessons Learned

- **Audit logs are not optional**: A Kubernetes cluster without audit logs is flying blind. If the cluster is compromised, you will have no way to determine the attack vector, entry point, or blast radius. This makes incident response and forensic analysis effectively impossible.

- **`Metadata` level is not enough for security**: The Kubernetes documentation recommends `Metadata` level for production, but for security-sensitive environments, `RequestResponse` is necessary. Without request bodies, you cannot see what the attacker actually did — you only see timestamps and resource names.

- **Static pods are a blind spot**: Any attacker with root access to a control plane node can deploy static pods that bypass the API Server entirely. Audit logging alone is insufficient — you need file integrity monitoring (FIM) on `/etc/kubernetes/manifests/` and strict SSH access controls.

- **Log shipping is as important as log generation**: Local storage with 7-day rotation means logs disappear within a week. If your control plane node crashes, you lose everything. Ship logs to centralized storage (S3, Elasticsearch, or a SIEM) with at least 1-year retention for compliance and forensics.

- **The API Server restart destroyed evidence**: During incident response, the team restarted kubelet and API Server as a first step — this rotated the active audit log file and destroyed the evidence. Incident response runbooks **must** include a step to preserve logs before restarting services.

- **SSH access to control plane is a privileged operation**: The attacker gained root access via SSH. Control plane nodes should have SSH access restricted via bastion hosts, SSH key-only authentication with hardware-backed keys, and at minimum multi-factor authentication.

- **Alert on log absence, not just log content**: An absence of audit events can indicate a misconfiguration or an attacker disabling logging. Alert when audit event volume drops to zero.

---

## Summary

The attack chain:

```
SSH brute force against control plane node cp-1 (root password guessed)
→ Attacker gains root shell on cp-1 at 23:30
→ Writes static pod manifest to /etc/kubernetes/manifests/kube-proxy-monitor.yaml
→ Kubelet detects new manifest and creates the pod — API Server is bypassed entirely
→ Pod runs cryptominer consuming 8 CPU + 6 GB RAM for 2 hours
→ CPUThrottlingHigh + NodePortExhaustion alerts fire at 03:14
→ On-call restarts API Server as first containment step — destroys active audit log
→ Security team discovers audit logs are empty for the breach window
→ Static pod manifest found on disk — confirms attack vector
→ But without audit logs: no RBAC audit trail, no API call history, no lateral movement detection
```

**Key metrics:**
- Time to detect: 3.7 hours (23:45 deployment → 03:14 alert)
- Time to contain: 30 minutes
- Forensic evidence available: 10% (only static pod manifest + auth.log entries)
- Impact: 60% cluster performance degradation, 12 worker nodes with port exhaustion, unknown data exfiltration risks

**What would have helped:**
- `RequestResponse` audit level for all critical resources
- Centralized log shipping to S3 with 365-day retention
- File integrity monitoring on `/etc/kubernetes/manifests/`
- Bastion host + MFA for SSH access to control plane
- Incident response runbook: **preserve logs before restarting any component**

The cluster had audit logging enabled — but it was "checklist compliance" logging, not "security" logging. The difference cost the team their only chance at understanding how the attacker got in, what they did, and whether they were still inside.
