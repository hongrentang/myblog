---
title: "Disk Full Causing Node NotReady: How Unrotated Logs Took Down a Kubernetes Node"
date: 2026-05-19
weight: 100010
slug: "disk-full-node-notready"
tags: ["kubernetes", "storage", "troubleshooting"]
categories: ["Storage"]
description: "Kubernetes Node NotReady troubleshooting — from misdiagnosing Calico to finding the root cause was a full root disk"
keywords: "kubernetes node not ready disk pressure, k8s node disk full, pod evicted disk pressure, container log rotation"
draft: false
featured: true
cover:
  image: "/images/disk-full-banner.svg"
  caption: "Disk Full — Node NotReady Troubleshooting"
---

# Disk Full Causing Node NotReady

## Common Search Queries

- kubernetes node not ready fix
- k8s node disk full root cause
- pod evicted due to disk pressure
- container log rotation kubernetes
- crictl prune unused images

---

## The Incident

**Environment**: K8S v1.28, containerd 1.7, 3 Master + 5 Worker, 100GB root disk.

**Time**: 04:10 AM, monitoring alert fired.

**Symptoms**: Worker node `node-03` went `NotReady`. All pods on it were `Evicted`.

```bash
kubectl get nodes
NAME      STATUS     ROLES    AGE
node-01   Ready      worker   180d
node-02   Ready      worker   180d
node-03   NotReady   worker   180d    # ← trouble
node-04   Ready      worker   60d
node-05   Ready      worker   60d
```

**Impact**: 30+ pods went down on node-03, including 2 critical API instances.

---

## Background

At 4 AM, the pager went off. Node-03 was NotReady. I immediately suspected kubelet or Calico — we'd had a Calico tunnel flapping incident before. I was about to drain the node and restart Calico.

Good thing I checked the monitoring dashboard first.

---

## Investigation

### Step 1: Check Node Conditions

```bash
kubectl describe node node-03
```

One line jumped out:

```
Conditions:
  Type                 Status  LastHeartbeatTime
  ----                 ------  -----------------
  NetworkUnavailable   False   ...
  MemoryPressure       False   ...
  DiskPressure         True    ...             # ← Here it is
  PIDPressure          False   ...
  Ready                Unknown ...
```

`DiskPressure: True`. The node was running out of disk space. My Calico theory was completely wrong.

### Step 2: SSH into the Node and Check Disk

```bash
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        98G   98G     0G 100% /             # ← Root disk full!
/dev/vdb1       500G  200G  300G  40% /data
```

Root disk 100% full. kubelet and containerd couldn't write logs, so the node declared itself unhealthy.

### Step 3: Find the Culprit

```bash
du -sh /var/log /var/lib/containerd /var/lib/kubelet 2>/dev/null | sort -rh
38G     /var/lib/kubelet
25G     /var/log
12G     /var/lib/containerd
```

`/var/log` was 25GB. Digging deeper:

```bash
ls -lh /var/log/pods/ingress-nginx_ingress-nginx-controller-*/ | head -10
total 12G
-rw-r----- 1 root root 2.0G May 19 02:30 0.log
-rw-r----- 1 root root 2.0G May 19 02:15 1.log
-rw-r----- 1 root root 2.0G May 19 02:00 2.log
```

2GB log files with no rotation in sight. The Nginx ingress controller had been writing to these files for months.

---

## Root Cause

When this node was provisioned, logrotate was never configured for the container logs under `/var/log/pods/`. Over three months, the ingress controller logs accumulated until they filled the root disk.

**Misdiagnosis**: I went straight to "kubelet or Calico is down" because that's what bit us last time. I didn't even glance at disk usage — I assumed 100GB was more than enough for system logs.

---

## Resolution

### Emergency Recovery (5 minutes)

```bash
# Prune unused container images
crictl rmi --prune

# Vacuum systemd journal
journalctl --vacuum-size=1G

# Truncate oversized pod logs
find /var/log/pods -name "*.log" -size +1G -exec truncate -s 0 {} \;
```

After clearing ~30GB:

```bash
df -h
/dev/vda1        98G   68G   30G  70% /     # Back to 70%
```

Within minutes the node returned to Ready, and evicted pods were rescheduled.

### Temporary Fix

```bash
cat > /etc/logrotate.d/containers << 'EOF'
/var/log/pods/*/*.log {
    rotate 7
    daily
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    maxsize 500M
}
EOF
```

### Long-Term Prevention

1. **Monitor disk usage**: Prometheus + Alertmanager at 80% threshold
2. **Kubelet eviction thresholds**: Reserve space for system and kubelet operations

```bash
# /var/lib/kubelet/config.yaml
evictionHard:
  imagefs.available: 15%
  memory.available: 500Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

3. **External log shipping**: Use Filebeat / Fluentd to forward logs off the node — don't keep them locally long-term

---

## Lessons Learned

- **NotReady → check Conditions first**: `kubectl describe node` tells you `DiskPressure` directly. Way faster than guessing
- **`crictl rmi --prune` is a lifesaver**: containerd often accumulates tens of GB of dangling images
- **Logrotate is NOT automatic in containers**: Kubelet has its own `containerLogMaxSize` and `containerLogMaxFiles` settings — configure them, don't rely on system logrotate

```yaml
# Kubelet container log rotation
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
containerLogMaxSize: 100Mi
containerLogMaxFiles: 5
```

- **100GB is not enough for a root disk**: Use at least 200GB, or mount /var/lib on a dedicated data volume

---

## Summary

The investigation chain:

```
Alert → node-03 NotReady
  → Misdiagnosed Calico (ready to restart it)
  → Saw Conditions: DiskPressure
  → SSH'ed in, df -h confirmed root disk 100%
  → du found /var/log at 25GB
  → Logrotate was never configured
  → crictl + journalctl + truncate cleaned up
  → Node recovered to Ready
```

Total time: 12 minutes. Five of those were wasted on the wrong hypothesis. Check node conditions first — they're there for a reason.
