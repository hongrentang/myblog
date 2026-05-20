---
title: "ETCD Space Full Brought Down the Cluster — 90 Minutes Without kubectl"
date: 2026-05-20
weight: 100080
slug: "etcd-space-full-cluster-outage"
tags: ["kubernetes", "etcd", "troubleshooting"]
categories: ["K8S"]
description: "Full postmortem of an ETCD space quota exceeded outage — from misdiagnosing API Server to finding the NOSPACE alarm"
keywords: "etcd空间满, etcd quota exceeded, kubernetes集群不可用, etcd NOSPACE, etcd compaction, kubectl无法使用"
draft: false
featured: true
cover:
  image: "/images/etcd-space-full-banner.svg"
  caption: "ETCD Space Full Cluster Outage"
---

# ETCD Space Full Brought Down the Cluster — 90 Minutes Without kubectl

## Background

Wednesday afternoon. Alerts started pouring in — not disk alerts, not pod alerts — **K8S API Server unreachable**.

```bash
kubectl get nodes
```

```
E0520 14:32:15.123456   12345 memcache.go:265] couldn't get current server API group list
E0520 14:32:15.123789   12345 memcache.go:265] Get "https://api.k8s-cluster.example.com:6443/api?timeout=32s": dial tcp: connect: connection refused
```

kubectl couldn't reach the API Server. Try again:

```bash
kubectl get pods -A
```

```
The connection to the server api.k8s-cluster.example.com was refused - did you specify the right host or port?
```

CI/CD all broken. New services couldn't deploy. Core services that depend on the API Server for service discovery were already reporting errors.

## Investigation

### Wrong Turn 1: Assumed API Server Was Down

kubectl can't connect — first thought: "API Server is down, restart it."

I SSH'd to the master node and checked kube-apiserver:

```bash
ssh master-01
crictl ps | grep kube-apiserver
```

```
f3a2b1c0d1e2   3 minutes ago   Running   kube-apiserver
```

Running? Then why can't kubectl connect?

```bash
# Check API Server logs
crictl logs --tail 50 f3a2b1c0d1e2
```

```
W0520 14:28:01.123456   12345 dispatcher.go:201] Failed to update etcd members list: clientv3: etcdserver: request timed out
W0520 14:28:11.123456   12345 dispatcher.go:201] Failed to update etcd members list: clientv3: etcdserver: request timed out
```

API Server was spamming etcd timeout errors. Not an API Server problem — **etcd was the issue**.

**Lesson**: When kubectl can't connect and the API Server is running, don't restart it. The API Server is stateless — restarting won't help if etcd is broken. Check the API Server logs first.

### Wrong Turn 2: Thought etcd Nodes Were Down

If etcd is timing out, are the etcd nodes dead?

```bash
crictl ps | grep etcd
```

```
d4e5f6a7b8c9   3 hours ago   Running   etcd
```

Also running. Check the logs:

```bash
crictl logs --tail 50 d4e5f6a7b8c9
```

```
{"level":"warn","msg":"rejected connection","remote-addr":"10.0.0.1:43876","error":"etcdserver: mvcc: database space exceeded"}
{"level":"warn","msg":"rejected connection","remote-addr":"10.0.0.2:43877","error":"etcdserver: mvcc: database space exceeded"}
```

**"database space exceeded"** — etcd hit its quota. It entered protection mode, rejecting all write operations.

```bash
# Check etcd storage status
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status --write-out=table
```

```
+------------------+----------------+---------+---------+-----------+-----------+------------+
|    ENDPOINT      |    DB SIZE     |  IN USE  |  RAFT TERM | RAFT INDEX | IS LEADER | ERR MSGS  |
+------------------+----------------+---------+---------+-----------+-----------+------------+
| https://127.0.0.1:2379 | 2.1 GB | 2.0 GB | 123456 | 987654 | true | "NOSPACE" |
+------------------+----------------+---------+---------+-----------+-----------+------------+
```

DB SIZE: 2.1 GB. The default `--quota-backend-bytes` is 2 GB. Hit the ceiling.

```bash
ETCDCTL_API=3 etcdctl ... alarm list
```

```
memberID: 1234567890 alarm: NOSPACE
```

**NOSPACE**. This is a protection mechanism — etcd detected the DB file exceeded the quota and blocked all writes to prevent an irreversible disk crash.

### Wrong Turn 3: Compacted But Didn't Defrag

Found the issue — space full. First instinct: compact.

```bash
# Get current revision
ETCDCTL_API=3 etcdctl ... endpoint status -w json | jq -r '.[].Status.revision'
```

```
8234567
```

```bash
# Compact all historical versions
ETCDCTL_API=3 etcdctl ... compact 8234567
```

```
compacted revision 8234567
```

Compaction succeeded. Then I made my second mistake — **I stopped there**.

```bash
kubectl get nodes
```

Still connection refused.

**Lesson**: compaction marks old versions as reclaimable, **but doesn't free disk space**. The space stays in the etcd DB file until you run `defrag`. I didn't know about defrag at the time and wasted another 15 minutes.

## Root Cause

etcd is an MVCC database — every modification to a key creates a new version. Old versions accumulate silently. In a busy cluster with frequent resource updates (events, endpointslices, configmaps), historical versions can quickly consume the entire quota.

```bash
# Check what consumes the most keys
ETCDCTL_API=3 etcdctl ... get / --prefix --keys-only | awk -F '/' '{print $2}' | sort | uniq -c | sort -rn | head -5
```

```
   2345678 /registry/events
    987654 /registry/endpointslices
    456789 /registry/configmaps
```

Events alone accounted for nearly half the data. The event TTL mechanism deletes the key, but etcd's MVCC keeps the historical version.

```
- Direct cause: etcd DB file hit the 2 GB `quota-backend-bytes` limit, triggering NOSPACE protection
- Root cause: No auto-compaction or defrag was configured. Historical versions (especially events and endpointslices) accumulated over a year
- Trigger: The cluster had been running for 456 days without any etcd maintenance
```

## Solution

### Emergency Recovery: Compact + Defrag

**Step 1: Compact historical versions**

```bash
REV=$(ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint status -w json | jq -r '.[].Status.revision')

ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  compact "$REV"
```

**Step 2: Defrag (releases actual disk space)**

```bash
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  defrag
```

Repeat defrag for each etcd member.

**Step 3: Disarm the alarm**

```bash
ETCDCTL_API=3 etcdctl ... alarm disarm
```

**Step 4: Verify recovery**

```bash
ETCDCTL_API=3 etcdctl ... endpoint status --write-out=table
```

```
+------------------+----------------+---------+---------+-----------+-----------+----------+
|    ENDPOINT      |    DB SIZE     |  IN USE  |  RAFT TERM | RAFT INDEX | IS LEADER | ERR MSGS |
+------------------+----------------+---------+---------+-----------+-----------+----------+
| https://127.0.0.1:2379 | 1.1 GB | 1.1 GB | 123456 | 987789 | true | "" |
+------------------+----------------+---------+---------+-----------+-----------+----------+
```

DB SIZE dropped from 2.1 GB to 1.1 GB. ERR MSGS empty.

```bash
kubectl get nodes
```

```
NAME       STATUS   ROLES                  AGE   VERSION
master-01  Ready    control-plane,master   456d v1.28.0
worker-01  Ready    <none>                 456d v1.28.0
worker-02  Ready    <none>                 456d v1.28.0
```

✅ **Recovery verified**:
- `kubectl get nodes` returns normally
- `etcdctl alarm list` is empty
- API Server logs show no more etcd timeouts
- CI/CD pipeline resumes

### Long-Term Fix

```bash
# 1. Enable auto-compaction (revision mode)
# Edit etcd static pod manifest: /etc/kubernetes/manifests/etcd.yaml
```

```yaml
spec:
  containers:
  - command:
    - etcd
    - --auto-compaction-mode=revision
    - --auto-compaction-retention=1000
```

```bash
# 2. Schedule periodic defrag via CronJob
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-defrag
  namespace: kube-system
spec:
  schedule: "0 3 * * 0"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-defrag
            image: bitnami/etcd:3.5
            command:
            - etcdctl
            - --endpoints=https://etcd:2379
            - --cacert=/etc/etcd/ca.crt
            - --cert=/etc/etcd/server.crt
            - --key=/etc/etcd/server.key
            - defrag
            volumeMounts:
            - mountPath: /etc/etcd
              name: etcd-certs
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
EOF
```

**Auto-compaction prevents accumulation; defrag releases space. Neither alone is sufficient.**

## What I Learned

1. **kubectl down ≠ API Server down ≠ etcd down**. The debug chain is kubectl → API Server → etcd. When API Server logs show etcd timeouts, go straight to etcd. Restarting the API Server won't fix it.

2. **"database space exceeded" is a protection mechanism, not a crash.** etcd deliberately rejects writes when the DB exceeds quota to prevent an unrecoverable disk-full scenario. But if you don't have compact + defrag configured, you *will* hit it eventually.

3. **compact ≠ defrag**. Compaction marks old data as reclaimable. Defrag actually frees the space back to the OS. From 2.1 GB to 1.1 GB — that 1 GB difference is historical versions that compaction *marked* but only defrag *released*.

4. **Events are the silent killer of etcd storage.** A busy cluster generates orders of magnitude more event revisions than any other resource. Events have TTL-based deletion, but etcd's MVCC keeps the old versions. Either enable auto-compaction or reduce `--event-ttl`.

5. **Clusters running over a year need etcd maintenance.** Most clusters ship with default etcd config — auto-compaction off, no defrag schedule. By the time you hit NOSPACE, it's an emergency. A weekly defrag CronJob costs almost nothing but prevents a major outage.
