---
title: "Kubernetes Node NotReady — When Your Node Dies and Nobody Tells You Why"
date: 2026-06-08
weight: 100420
slug: "node-notready-investigation"
tags: ["kubernetes", "troubleshooting", "node", "kubelet", "network"]
categories: ["Troubleshooting"]
description: "A Kubernetes node NotReady incident — a systematic investigation of why three worker nodes went NotReady simultaneously, covering kubelet hang, CNI failures, disk pressure, and container runtime deadlocks"
keywords: "kubernetes node notready, kubelet notready, node status unknown, kubectl get nodes notready, kubernetes node troubleshooting"
draft: false
featured: true
cover:
  image: ""
  caption: "Node NotReady — Troubleshooting"
---

## Common Search Queries

| Query | Intent |
|---|---|
| `kubectl get nodes notready` | Check which nodes are unhealthy |
| `kubernetes node status unknown` | Understand node lifecycle states |
| `kubelet not reporting status` | Debug kubelet heartbeat failures |
| `node.kubernetes.io/unreachable:NoSchedule` | Taint-based pod eviction |
| `kubernetes node disk pressure` | Node resource pressure conditions |
| `crictl ps not working` | Container runtime deadlock |
| `calico node crash loop back off` | CNI plugin failures |
| `kubelet hung d state` | Kubelet stuck in uninterruptible sleep |
| `kubernetes node-monitor-grace-period` | Controller manager node monitoring |
| `kubelet volume mount stuck` | PVC mount causing kubelet hang |

---

## The Incident

### Environment

| Component | Version / Detail |
|---|---|
| Kubernetes | v1.28 |
| Nodes | 30 worker nodes, 3 control-plane nodes |
| Pod count | ~500 pods |
| CNI | Calico (v3.26) |
| Container Runtime | containerd 1.6.x |
| OS | Ubuntu 22.04 LTS |
| Cloud | On-premise bare-metal |
| Monitoring | Prometheus + Grafana + PagerDuty |

### Timeline

It was 03:17 AM on a Tuesday morning. The on-call engineer's phone lit up with a PagerDuty alert: **3 nodes are NotReady**. Within minutes, the alert cascade began — pods on the affected nodes were stuck in `Terminating` state, and the remaining 27 nodes suddenly had to absorb the rescheduled workloads, pushing CPU and memory allocation well past 85%.

The initial symptoms were:

- `kubectl get nodes` showed three workers as `NotReady`
- All pods on those nodes stayed in `Terminating` or `Unknown` state
- The remaining nodes showed elevated memory pressure
- Several critical service SLIs started degrading
- PagerDuty was firing alerts every 30 seconds

```bash
# What the screen looked like at 03:17
$ kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
node-01         Ready      <none>   342d   v1.28.4
node-02         Ready      <none>   342d   v1.28.4
node-03         Ready      <none>   342d   v1.28.4
node-04         NotReady   <none>   342d   v1.28.4   # <-- BAD
node-05         NotReady   <none>   342d   v1.28.4   # <-- BAD
node-06         NotReady   <none>   342d   v1.28.4   # <-- BAD
node-07         Ready      <none>   342d   v1.28.4
...
```

---

## Background

### Kubernetes Node Lifecycle

Understanding why a node goes `NotReady` requires knowing how the control plane tracks node health. The mechanism has three components:

1. **kubelet heartbeats** — Every `--node-status-update-frequency` (default 10s), the kubelet on each node posts a `NodeStatus` update to the API server. This includes conditions like `Ready`, `DiskPressure`, `MemoryPressure`, `PIDPressure`, and `NetworkUnavailable`.

2. **controller-manager monitoring** — The `node-controller` in `kube-controller-manager` checks the timestamp of each node's last heartbeat. If it hasn't been updated within `--node-monitor-grace-period` (default 40s), the controller marks the node as `Unhealthy`. If it exceeds `--node-startup-grace-period` (default 1m0s) for a newly joining node, or `--node-monitor-grace-period` for an existing node, the condition transitions to `Unknown`.

3. **Taint-based eviction** — Once a node is `NotReady` or `Unreachable`, the controller-manager applies the taint `node.kubernetes.io/unreachable:NoSchedule` (or `node.kubernetes.io/not-ready:NoSchedule`), preventing new pods from scheduling onto the node. After `--pod-eviction-timeout` (default 5m), pods on the affected node are evicted and rescheduled elsewhere.

### Node Conditions

Kubernetes tracks five node conditions:

| Condition | Description | Common Causes |
|---|---|---|
| `Ready` | Node is healthy and accepting pods | Kubelet down, CNI broken, runtime deadlock |
| `DiskPressure` | Disk space on the node is low | Log rotation failure, image pull flood |
| `MemoryPressure` | Node memory is low | Workload overcommit, memory leak |
| `PIDPressure` | Too many processes on the node | Fork bomb, zombie processes |
| `NetworkUnavailable` | Node network is misconfigured | CNI plugin crash, iptables corruption |

---

## Investigation

### Step 1: Check Node Status

The first command is always obvious:

```bash
kubectl get nodes
kubectl describe node node-04
```

The `describe node` output contained the critical clue in the `Conditions` section:

```
Conditions:
  Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----                 ------  -----------------                 ------------------                ------                       -------
  NetworkUnavailable   False   Mon, 08 Jun 2026 03:10:17 +0000  Mon, 08 Jun 2026 02:55:03 +0000   CalicoIsUp                   Calico is running on this node
  MemoryPressure       False   Mon, 08 Jun 2026 03:10:17 +0000  Tue, 02 Jun 2026 10:22:14 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure         True    Mon, 08 Jun 2026 03:10:17 +0000  Mon, 08 Jun 2026 03:01:44 +0000   KubeletHasDiskPressure       kubelet has disk pressure
  PIDPressure          False   Mon, 08 Jun 2026 03:10:17 +0000  Tue, 02 Jun 2026 10:22:14 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready                Unknown Mon, 08 Jun 2026 03:10:17 +0000  Mon, 08 Jun 2026 03:16:47 +0000   NodeStatusUnknown            Kubelet stopped posting node status.
```

Three things jumped out immediately:
1. **Ready = Unknown** with `Kubelet stopped posting node status` — the controller-manager had not received a heartbeat.
2. **DiskPressure = True** with a very recent transition time.
3. The last heartbeat was at 03:10:17, but the transition to Unknown happened at 03:16:47 — roughly 6.5 minutes later, well past the 40-second grace period.

### Step 2: SSH and Check Kubelet

```bash
# SSH to the affected node
ssh node-04

# Check kubelet status
systemctl status kubelet
```

Output:
```
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2026-06-08 02:55:03 UTC; 24min ago
     Docs: https://kubernetes.io/docs/
 Main PID: 1837 (kubelet)
   Tasks: 23 (limit: 49152)
   Memory: 312.4M
   CGroup: /system.slice/kubelet.service
           └─1837 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml
```

Kubelet was **running** — not crashed, not stopped. So why wasn't it sending heartbeats?

```bash
journalctl -u kubelet --since "10 min ago" | tail -50
```

Key log entries:

```
Jun 08 03:15:01 node-04 kubelet[1837]: E0608 03:15:01.234567   1837 kubelet_node_status.go:92] "Failed to update node status" err="Post \"https://api-server:6443/...\": context deadline exceeded (Client.Timeout exceeded while awaiting headers)"
Jun 08 03:15:01 node-04 kubelet[1837]: E0608 03:15:01.234678   1837 kubelet_node_status.go:96] "Node status update failed, will retry"
Jun 08 03:14:58 node-04 kubelet[1837]: I0608 03:14:58.123456   1837 reconciler.go:224] "VolumeReconciler: Volume operation stuck" volume="pvc-xxxxx" plugin="kubernetes.io/aws-ebs" state="mount" duration="5m32s"
```

The last line was the first real clue: the **volume reconciler was stuck** on a mount operation.

### Step 3: Check Syslog for Kubelet Entries

```bash
grep -i "notready\|node.status\|heartbeat\|volume" /var/log/syslog | tail -20
```

Output revealed a pattern — repeated `context deadline exceeded` errors from the kubelet's API client, all concurrent with a mount operation on a specific PVC that would not complete.

```
Jun 08 03:10:01 node-04 kubelet[1837]: I0608 03:10:01.123456   1837 operation_executor.go:110] "MountVolume operation started" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
Jun 08 03:10:01 node-04 kubelet[1837]: I0608 03:10:01.123789   1837 operation_executor.go:115] "MountVolume.WaitForAttach" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
Jun 08 03:10:01 node-04 kubelet[1837]: I0608 03:10:01.124000   1837 operation_executor.go:120] "MountVolume.MountDevice" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# ... no further progress on this mount ...
Jun 08 03:14:58 node-04 kubelet[1837]: I0608 03:14:58.123456   1837 reconciler.go:224] "VolumeReconciler: Volume operation stuck" volume="pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### Step 4: Check Container Runtime

```bash
crictl pods | head -10
```

The command **hung** — it did not return for over 30 seconds. This was a red flag.

```bash
crictl ps -a | head -10
```

This also hung. The containerd socket was not responding:

```bash
ls -la /run/containerd/containerd.sock
# Socket exists but operations time out
```

Checking containerd:

```bash
systemctl status containerd
```

```
● containerd.service - containerd container runtime
   Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2026-06-08 02:55:03 UTC; 24min ago
     Docs: https://containerd.io/
 Main PID: 1521 (containerd)
   Tasks: 45
   Memory: 1.2G
   CGroup: /system.slice/containerd.service
```

Containerd was running, but its gRPC socket was not responding. The high memory usage (1.2 GB) suggested containerd was under significant load, likely from handling numerous container operations triggered by the crash-looping CNI.

### Step 5: Check CNI (Calico)

From a healthy node (or a management box):

```bash
kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
```

```
NAME                READY   STATUS             RESTARTS   AGE   IP              NODE
calico-node-abc12   1/1     Running            0          10d   192.168.1.10    node-01
calico-node-def34   0/1     CrashLoopBackOff   47         24m   192.168.1.11    node-04   # <--
calico-node-ghi56   0/1     CrashLoopBackOff   51         24m   192.168.1.12    node-05   # <--
calico-node-jkl78   0/1     CrashLoopBackOff   43         24m   192.168.1.13    node-06   # <--
calico-node-mno90   1/1     Running            0          10d   192.168.1.14    node-02
```

All three NotReady nodes had Calico pods in `CrashLoopBackOff`. Checking the Calico logs:

```bash
kubectl logs -n kube-system calico-node-def34 --previous | tail -30
```

Key error:

```
2026-06-08 03:05:23.456 [ERROR][1] felix/iptables.go:789: Failed to update iptables rules: error executing iptables-restore: exit status 4 (iptables-restore: line 47: COMMAND_FAILED)
2026-06-08 03:05:23.789 [ERROR][1] felix/iptables.go:790: iptables-save output: # Warning: iptables-legacy tables present, use iptables-legacy-save to see them
2026-06-08 03:05:23.790 [ERROR][1] felix/iptables.go:791: Setting iptables to legacy mode -- iptables-legacy...
```

This pointed to **iptables rule corruption**. Checking the node directly:

```bash
iptables -L -t nat | head -20
```

The output was garbled — iptables rules were in an inconsistent state, with overlapping entries from concurrent modification attempts.

```bash
# Check which process is modifying iptables concurrently
auditctl -w /sbin/iptables -p x -k iptables_changes
ausearch -k iptables_changes --start today | tail -20
```

The audit logs revealed that both `calico-node` (Felix) and another system tool (a legacy firewall management agent) were writing iptables rules simultaneously, causing race conditions.

### Step 6: Check Disk Pressure

```bash
df -h /var/lib/kubelet
df -h /
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        98G   96G   2G   98% /           # <-- ROOT PARTITION ALMOST FULL
/dev/sdb1       500G  120G  380G  24% /var/lib/kubelet
```

The root partition was at 98% capacity. The disk pressure condition was real.

```bash
du -sh /var/log/journal/
# 12G     /var/log/journal/
```

Journal logs had consumed 12 GB of the root partition. The crash-looping Calico pods were generating massive log output:

```bash
journalctl --disk-usage
# Archived and active journals take up 12.0G in the file system.
```

The container log rotation had also failed:

```bash
ls -la /var/log/pods/
# Some log files dated days ago — rotation was not happening
```

Checking kubelet eviction config:

```bash
cat /var/lib/kubelet/config.yaml | grep -i "eviction"
```

```
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
```

The `nodefs.available: 10%` threshold was breached (2% free), triggering the `DiskPressure` condition.

### Step 7: Check Kubelet Configuration

```bash
kubelet --version
# Kubernetes v1.28.4

cat /var/lib/kubelet/config.yaml | grep -i "nodeStatusUpdateFrequency\|heartbeat"
```

```
nodeStatusUpdateFrequency: 10s
```

The default 10-second update frequency meant the kubelet should have been sending heartbeats every 10 seconds. But the stuck volume mount had caused the kubelet's internal API client to block, preventing any status updates from being sent.

```bash
# Check for D-state processes (uninterruptible sleep)
ps aux | grep " D"
```

```
root      1837  0.3  0.2  987654 32100 ?        D    03:10   2:34 /usr/bin/kubelet --config=/var/lib/kubelet/config.yaml
root      6201  0.0  0.0      0     0 ?        D    03:10   0:00 [kworker/u4:2]
```

The kubelet process was in **D-state** (uninterruptible sleep), indicating it was blocked on an I/O operation — in this case, the stuck NFS-backed PVC mount.

---

## Root Cause

The incident was caused by **three concurrent issues** that converged simultaneously:

### Primary: Kubelet Hang on Volume Mount

A pod using an **NFS-backed PVC** had been scheduled to node-04. The underlying NFS server became unresponsive (later traced to a network switch firmware issue affecting a specific VLAN). The kubelet's volume reconciler initiated the mount and blocked in **D-state** (uninterruptible sleep) while waiting for the NFS server to respond.

Since the kubelet uses a single-threaded event loop for status updates (in versions prior to the improved volume handling in later releases), the blocked mount operation prevented the heartbeat goroutine from posting status to the API server.

### Secondary: Calico CNI Crash-Looping

On all three affected nodes, Calico's Felix agent was crash-looping due to **iptables rule corruption**. Investigation revealed that a legacy firewall management agent (leftover from a previous security tool deployment) was also modifying iptables rules on a cron schedule. When Calico and the legacy tool attempted concurrent iptables operations, the iptables kernel lock was contended, causing `iptables-restore` to fail with `COMMAND_FAILED`.

Each Calico crash triggered a restart, and the crash-looping generated massive log output — contributing to the disk pressure issue.

### Tertiary: Disk Pressure from Log Rotation Failure

The root partition (`/dev/sda1` at 98%) was the final trigger. The crash-looping Calico pods filled the journal with error messages. Simultaneously, container log rotation had stalled because:

- The `kubelet-eviction` log was also growing rapidly
- `journald` had no size limit configured (default behavior consumes up to 10% of the filesystem)
- The `DiskPressure` condition was set when `nodefs.available` dropped below 10%

### The Chain Reaction

```
03:10:00 -- NFS server becomes unresponsive
03:10:01 -- Kubelet begins volume mount, enters D-state
03:10:01 -- Heartbeat from kubelet stops
03:10:17 -- Last successful NodeStatus update received by API server
03:10:?? -- Calico iptables update collides with legacy tool, Felix crashes
03:11:00 -- Calico crash-loop begins, journal fills with errors
03:12:00 -- Containerd starts responding slowly (high memory from crash-loop operations)
03:13:00 -- CRI operations start timing out
03:14:00 -- DiskPressure detected (root partition at 95%+)
03:14:58 -- Kubelet volume reconciler reports stuck operation
03:16:47 -- Node controller marks nodes Unknown (node-monitor-grace-period expired)
03:16:48 -- Taint node.kubernetes.io/unreachable:NoSchedule applied
03:17:00 -- PagerDuty alert fires
03:17:?? -- Pods on affected nodes marked Terminating, rescheduled to remaining nodes
03:20:00 -- Remaining nodes reach 85%+ resource utilization, cascading pressure begins
```

---

## Resolution

### Emergency Response (Immediate)

Step 1: **Restart kubelet** on all three affected nodes

```bash
for node in node-04 node-05 node-06; do
    ssh $node "systemctl restart kubelet"
done
```

This broke the D-state hang. The kubelet processes were terminated by systemd and restarted cleanly. On restart, the volume reconciler retried the mount, and since the NFS issue was independently resolved (the network team fixed the switch port), the mounts succeeded.

```bash
# Verify kubelet is posting heartbeats again
kubectl get nodes
# All nodes should show Ready within 1-2 minutes
```

Step 2: **Force delete stuck pods**

```bash
# Identify pods stuck in Terminating on affected nodes
kubectl get pods --all-namespaces --field-selector spec.nodeName=node-04 | grep Terminating

# Force delete them
kubectl delete pod --force --grace-period=0 -n <namespace> <pod-name>
```

Step 3: **Clear journal logs** to relieve disk pressure

```bash
for node in node-04 node-05 node-06; do
    ssh $node "journalctl --vacuum-time=1d"
done
```

This freed approximately 8 GB of space on the root partition.

```bash
# Verify disk pressure is relieved
df -h /
```

Step 4: **Fix Calico** by restarting the DaemonSet pods

```bash
kubectl delete pod -n kube-system -l k8s-app=calico-node --force --grace-period=0
```

Also, disable the legacy firewall agent that was causing iptables conflicts:

```bash
for node in node-04 node-05 node-06; do
    ssh $node "systemctl stop legacy-firewall-agent && systemctl disable legacy-firewall-agent"
done
```

Step 5: **Drain and uncordon** nodes after recovery

```bash
kubectl uncordon node-04
kubectl uncordon node-05
kubectl uncordon node-06

# Verify pods are scheduling onto recovered nodes
kubectl get pods -o wide | grep node-04
```

### Long-Term Fixes

| # | Action | Configuration |
|---|---|---|
| 1 | **Add node condition monitoring alerts** for all 5 conditions | Prometheus: `kube_node_status_condition{condition="Ready",status="true"} == 0` |
| 2 | **Configure disk pressure eviction** | Set `evictionHard` in kubelet config: `nodefs.available: 5%`, `imagefs.available: 10%` |
| 3 | **Limit journald log size** | `/etc/systemd/journald.conf`: `SystemMaxUse=5G`, `MaxFileSec=7day` |
| 4 | **Use PodDisruptionBudget** for critical workloads | `minAvailable: 2` or `maxUnavailable: 1` |
| 5 | **Add kubelet liveness monitoring** | Monitor `http://<node>:10248/healthz` with external health checks |
| 6 | **Remove legacy iptables management tooling** | Decommission `legacy-firewall-agent` across all nodes |
| 7 | **Implement chaos engineering tests** | Regular chaos experiments: node failure, network partition, disk pressure, CNI kill |
| 8 | **Set pod eviction timeout** | Review `--pod-eviction-timeout` (default 5m) — potentially reduce for critical workloads |

### Kubelet Configuration Hardening

```yaml
# /var/lib/kubelet/config.yaml additions
evictionHard:
  nodefs.available: 5%
  imagefs.available: 10%
  nodefs.inodesFree: 5%
evictionSoft:
  nodefs.available: 10%
evictionSoftGracePeriod:
  nodefs.available: 2m0s
evictionMaxPodGracePeriod: 60s
```

### Journald Configuration

```ini
# /etc/systemd/journald.conf
[Journal]
SystemMaxUse=5G
SystemKeepFree=2G
MaxFileSec=7day
RuntimeMaxUse=1G
```

---

## Lessons Learned

| Issue | How We Detected It | How We Fixed It | How We Prevent It |
|---|---|---|---|
| Kubelet hung in D-state on volume mount | `ps aux \| grep " D"` showed kubelet in uninterruptible sleep | `systemctl restart kubelet` | Add kubelet healthz endpoint monitoring; use `mount -t nfs -o hard,intr` for NFS; implement volume mount timeout |
| CNI crash-looping from iptables corruption | `kubectl get pods -n kube-system` showed CrashLoopBackOff on calico-node | Stopped legacy firewall agent, deleted Calico pods | Audit all nodes for competing iptables management tools; use `iptables-legacy` vs `iptables-nft` consistently |
| Disk pressure from journal logs | `df -h /` showed 98% usage; `journalctl --disk-usage` showed 12 GB | `journalctl --vacuum-time=1d` freed 8 GB | Limit journald with `SystemMaxUse=5G`; set up log rotation alerts; add log aggregation (Loki/Elasticsearch) |
| Containerd unresponsive | `crictl pods` hung indefinitely | Kubelet restart (which triggered containerd restart via CRI) | Set containerd memory limits; monitor containerd gRPC latency |
| Controller-manager marking nodes NotReady | Default `node-monitor-grace-period: 40s` | Already configured — worked as designed | Review and potentially adjust grace period; but this was working correctly |
| PagerDuty alert flooding | 30+ alerts in 5 minutes | Acknowledged, triaged, resolved | Set up alert deduplication and rate limiting in PagerDuty |
| Cascading resource pressure on remaining nodes | Prometheus showed 85%+ CPU/memory on surviving nodes | Incident resolved before cascade caused failures | Implement cluster-autoscaler; use topology spread constraints; set resource quotas per namespace |

---

## Summary

### Chain of Events (Visual)

```
NFS Server Failure
       │
       ▼
Kubelet Volume Mount (D-state) ──► Heartbeat Stops
       │
       ├──► Controller-manager: node-monitor-grace-period expires (40s)
       │         │
       │         ▼
       │    Node → NotReady / Unknown
       │         │
       │         ▼
       │    Taint: node.kubernetes.io/unreachable:NoSchedule
       │         │
       │         ▼
       │    Pods evicted → Rescheduled to healthy nodes
       │         │
       │         ▼
       │    Cascading resource pressure on remaining nodes
       │
       ├──► Calico Felix crashes (iptables race condition)
       │         │
       │         ▼
       │    CNI crash-loop → Journal fills → DiskPressure
       │
       └──► Containerd high memory → CRI operations timeout
```

### Key Takeaway

The Node NotReady incident was a **convergence of three independent failure modes** — a blocked volume mount, a CNI crash-loop from iptables corruption, and disk pressure from log overflow. Any single issue would have been manageable, but their combination overwhelmed the standard recovery mechanisms:

1. The **kubelet hang** prevented automatic heartbeat recovery
2. The **CNI crash-loop** consumed system resources and filled logs
3. The **disk pressure** prevented new log writes and container operations

### Actionable Advice

- **Monitor kubelet heartbeats** at the API server level — don't rely solely on `systemctl status` on the node
- **Set journald log limits** on every node — default unlimited journal growth will fill any partition
- **Audit for conflicting system agents** — multiple tools managing iptables, systemd services, or cron jobs often cause subtle failures
- **Test node recovery procedures** with chaos engineering — a kubelet restart is simple to test, but the chain of events that prevents successful recovery is what you need to discover before production
- **Use PodDisruptionBudgets** — they won't prevent node failures, but they will prevent the total workload collapse that happens when all pods evacuate simultaneously
- **Configure evictionHard thresholds** that give enough lead time before disk exhaustion

---

*Published: 2026-06-08 | Tagged: kubernetes, troubleshooting, node, kubelet, network | Blog: https://blog.777157.xyz*
