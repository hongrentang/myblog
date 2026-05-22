---
title: "NFS Hang Deep Dive — When All Processes Fall Into D State"
date: 2026-05-22
weight: 100150
slug: "nfs-hang-d-state-troubleshooting"
tags: ["nfs", "storage", "d-state", "linux", "troubleshooting"]
categories: ["存储"]
description: "When an NFS server dies and every process on the client freezes — from SSH being unreachable to finding D-state processes and force-recovering the node"
keywords: "nfs hang, d state, uninterruptible sleep, nfs hard mount, process hung, linux troubleshooting"
draft: false
featured: true
cover:
  image: "/images/nfs-hang-banner.svg"
  caption: "NFS Hang — D State Troubleshooting"
---

# NFS Hang Deep Dive — When All Processes Fall Into D State

## The Incident

Friday, 4:30 PM. Alarms explode.

A Java application server node has vanished — not "slow," completely unreachable. Every health probe on that node is timing out.

Try SSH:

```bash
ssh app-server-01
```

```
ssh: connect to host app-server-01 port 22: Connection timed out
```

Can't even SSH in. This isn't an application issue — this is the OS itself hanging.

Other monitoring shows replicas of the same service on healthy nodes are also starting to fail. Logs show:

```
java.io.IOException: No such device or address
   at java.base/sun.nio.ch.FileDispatcherImpl.write0(Native Method)
```

Another service reports:

```
Error syncing pod, skipping: failed to "PrepareDynamicResources" for "app-pod" 
  with PrepareDynamicResourcesError: "rpc error: code = Internal 
  desc = wait for remote storage condition: context deadline exceeded"
```

The kubelet on that node is unreachable too. Node status flips to NotReady.

```bash
kubectl get nodes
```

```
NAME            STATUS     ROLES    AGE
app-server-01   NotReady   worker   187d
app-server-02   Ready      worker   187d
app-server-03   Ready      worker   187d
```

**Impact**: 20+ Pods on the node are completely stuck. Some can't be rescheduled (local PV / hostPath dependencies). Core services degraded. And K8S can't properly evict pods from the node because kubelet itself is frozen.

## Investigation

### Wrong turn 1: Try SSH

SSH fails. Try with verbose to see where it hangs:

```bash
ssh -vvv app-server-01
```

```
debug1: Connecting to app-server-01 [10.0.0.10] port 22.
debug1: Connection established.
debug1: identity file /home/user/.ssh/id_rsa type 0
debug1: Local version string SSH-2.0-OpenSSH_8.2p1
# Then it hangs — nothing else
```

TCP connection succeeds (kernel networking is still alive), but SSH auth hangs — PAM tries to read `/home` (NFS-mounted) for the user's authorized_keys.

**Lesson learned**: The node had `/home` mounted over NFS (centralized user home directories). SSH's PAM module reads authorized_keys from the user's home directory — which lives on the NFS share. NFS down → SSH hangs. This isn't an SSH bug, it's an NFS chain reaction.

We eventually got in through out-of-band management (BMC/iDRAC):

```bash
# First thing to check — process states
ps aux | grep "^[RD]"
```

```
USER       PID %CPU %MEM    VSZ   RSS STAT START   TIME COMMAND
root      1234  0.0  0.1      0     0 D    Apr22   0:01 [kworker/0:0]
app       2345  0.0  0.2 423456 12345 D    15:30   0:15 java -jar app.jar
app       2346  0.0  0.1 123456  7890 D    15:31   0:10 nginx: worker process
root      3456  0.0  0.0      0     0 D    15:32   0:00 [nfsv4.0]
root      4567  0.0  0.0      0     0 D    15:33   0:00 [kworker/u4:0]
```

A wall of **D state** processes — Uninterruptible Sleep.

### Wrong turn 2: Blame DDoS or OOM

Seeing all those D state processes, my first thought was resource exhaustion.

```bash
dmesg | grep -i "out of memory"
```

No OOM records. Memory is fine:

```bash
free -h
```

```
              total        used        free      shared  buff/cache   available
Mem:           31Gi        18Gi         2Gi       1.0Gi        11Gi        12Gi
Swap:           2Gi       200Mi       1.8Gi
```

```bash
uptime
```

```
 16:45:00 up 187 days,  3:12,  0 users,  load average: 45.12, 30.21, 15.08
```

Load average 45 on a 16-core machine. But D state processes all count toward load. Very few processes are actually in R (runnable) state.

### Wrong turn 3: Try kill -9

Gut reaction — kill the stuck processes:

```bash
kill -9 2345
```

```
-bash: kill: (2345) - Operation not permitted
```

Wait, root can't kill it?

```bash
kill -9 3456
```

```
-bash: kill: (3456) - Operation not permitted
```

**Lesson learned**: D state (Uninterruptible Sleep) means the process is in a kernel operation (NFS I/O) that can't be interrupted. `kill -9` sends SIGKILL, but the kernel can't deliver it until the process returns to userspace — which it never will because it's stuck waiting for NFS.

This is the cruelest thing about D state: **you cannot kill it**. The signal is marked as pending, but the process won't die until the kernel I/O completes or times out.

For NFS with the default `hard` mount option, the client retries **indefinitely** — the D state processes will never recover on their own.

```bash
# Check mount options
mount | grep nfs
```

```
10.0.0.100:/data on /mnt/nfs type nfs4 (rw,relatime,vers=4.0,...,hard,proto=tcp,timeo=600,retrans=2,...)
```

There it is — **`hard`**. NFS will retry forever.

### The breakthrough: NFS server is dead

Check the NFS server:

```bash
ping 10.0.0.100
```

```
Request timeout for icmp_seq 0
Request timeout for icmp_seq 1
...
```

NFS server is completely gone.

```bash
# Check stuck NFS client state
cat /proc/2345/stack
```

```
[<ffffffffc06b3a40>] nfs4_wait_bit_killable+0x20/0x60 [nfsv4]
[<ffffffffc069e2a0>] __rpc_execute+0x250/0x3c0 [sunrpc]
[<ffffffffc069d3c0>] rpc_run_task+0x100/0x120 [sunrpc]
[<ffffffffc06b4810>] nfs4_call_sync_private+0x90/0xc0 [nfsv4]
[<ffffffffc06b8890>] _nfs4_do_open+0x390/0x650 [nfsv4]
[<ffffffffc06b8ff0>] nfs4_open+0xa0/0x120 [nfsv4]
[<ffffffffc069ae90>] nfs4_file_open+0x60/0xc0 [nfsv4]
[<ffffffff8121b340>] do_dentry_open+0x140/0x2e0
```

Call trace shows everything stuck at `nfs4_wait_bit_killable` → `__rpc_execute` — the kernel RPC layer waiting for the NFS server.

```bash
# Count D state processes
ps -eo stat | grep -c "^D"
```

```
23
```

23 processes in D state. Including Java main process, Nginx workers, sshd, kubelet, and kernel workers.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct | NFS storage server 10.0.0.100 went down (power failure — array didn't remount after restart) |
| Propagation | NFS `hard` mount causes client processes to enter D state with infinite retry |
| Blast radius | System daemons (sshd, kubelet) hit NFS paths, freezing the entire node |
| Recovery block | D state processes can't be killed; they only recover when NFS comes back or the node reboots |

**Why `hard` mode is dangerous**:

| Mount Option | Behavior When NFS Hangs |
|-------------|--------------------------|
| `hard` (default) | Process retries forever, stays in D state, never returns errors to the application |
| `soft` | Returns I/O error after `retrans` retries, application can catch and handle |

`hard` guarantees data won't silently corrupt (writes won't return success until they actually land), but the price is — **NFS goes down, the entire machine goes down**.

## Fix

### Option A: Wait for NFS recovery (passive, not always viable)

If NFS comes back quickly, D state processes recover automatically:

```bash
# After NFS recovers
ps aux | grep " D"
# D state count should drop
```

But the NFS server had a power failure — array recovery takes time. We couldn't wait.

### Option B: Force-unmount NFS (active, but risky)

```bash
# First try force unmount
umount -f /mnt/nfs
```

```
umount: /mnt/nfs: target is busy
```

```bash
# Find what's using the mount
fuser -m /mnt/nfs
```

```
/mnt/nfs:           1234 2345 2346 3456 4567
```

```bash
# Lazy unmount — remove from filesystem tree immediately
umount -l /mnt/nfs
```

`umount -l` removes the mount point from the VFS tree so new accesses can't reach NFS. But **already-stuck D state processes won't recover** — they're still waiting for the NFS RPC layer.

In some kernel versions, `umount -l` can leave the NFS state machine in an inconsistent state, making reboot harder. Use with caution.

### Option C: Reboot (nuclear, but must be done right)

Since SSH is unreachable, only BMC works:

```bash
ipmitool -H bmc-ip -U admin -P pass chassis power cycle
```

**But reboot might fail** — the system will try to sync filesystems during shutdown, which means writing to NFS. If NFS is still hung, shutdown itself can hang.

```bash
# This might not even work
reboot -f
```

If even `reboot -f` hangs, hard power cycle:

```bash
ipmitool -H bmc-ip -U admin -P pass chassis power off
sleep 30
ipmitool -H bmc-ip -U admin -P pass chassis power on
```

**What actually happened**: We hard-power-cycled via BMC. Post-reboot:

```bash
ps -eo stat | grep -c "^D"
```

```
0
```

```bash
kubectl get nodes
```

```
NAME            STATUS   ROLES    AGE
app-server-01   Ready    worker   187d
```

✅ Node recovered.

### Verification

```bash
# 1. Confirm zero D state processes
ps -eo stat,pid,comm | grep "^D"
# Empty

# 2. Kubelet healthy
systemctl status kubelet
# active (running)

# 3. Pods recovered
kubectl get pods -o wide | grep app-server-01
# All Running/Ready

# 4. Application health
curl http://app-service:8080/health
# 200 OK
```

### Long-term prevention

```bash
# 1. Use soft mount + application retry (for appropriate workloads)
# /etc/fstab
10.0.0.100:/data  /mnt/nfs  nfs4  soft,timeo=30,retrans=3,noac  0 0
```

```
mount -o remount /mnt/nfs
```

**Warning**: `soft` is not a silver bullet. For databases and critical writes, `soft` can cause silent data corruption (write returns success but data didn't actually land on disk). Use `hard` for those — but accept the tradeoff.

| Approach | Use Case | Pros | Cons |
|----------|----------|------|------|
| `soft` + app retry | Stateless services, cache | NFS failure doesn't freeze node | Silent write failures possible |
| `hard` + NFS HA | Databases, stateful services | Data consistency | Requires NFS HA architecture |
| Object storage API | New systems | Failure isolation | Larger app changes |
| K8S CSI + local storage | K8S workloads | Failure domain isolation | Not universal |

```bash
# 2. Don't mount critical system paths over NFS
# /home, /var/log, /var/run should be local
# If you must use centralized home dirs, use autofs (mounts on demand)

# 3. NFS HA is not optional for critical storage
# DRBD + Pacemaker / GlusterFS / CephFS

# 4. Monitor NFS connectivity
while true; do
  timeout 5 ls /mnt/nfs/.health 2>/dev/null || echo "NFS down: $(date)" >> /var/log/nfs_watchdog.log
  sleep 60
done
```

## What I Learned

1. **D state processes cannot be killed.** Never expect `kill -9` to work on them. You're sending a signal that gets marked as pending, but the process never returns to userspace to handle it. D state vs Z state (zombie): zombies CAN be reaped by their parent. D state processes can only be unblocked by the kernel I/O completing.

2. **`nfs hard` is a double-edged sword.** It prevents silent data corruption, but the price is "if NFS hangs, you can't even SSH in." Never mount critical system paths (`/home`, `/var/log`) over NFS on production servers — when NFS fails, you lose your debugging tools too.

3. **The scariest outage isn't error logs — it's losing SSH access.** Without BMC, we'd have had no idea what happened. Out-of-band management isn't optional for production.

4. **`/proc/<pid>/stack` is the D state debugging superpower.** When `kill -9` doesn't work, don't try again — read the stack. `nfs4_wait_bit_killable` immediately tells you it's NFS.

5. **NFS failures cascade in K8S.** Kubelet, CRI shim, monitoring agents — all can hit NFS-mounted paths. A single stuck NFS mount can cause kubelet to freeze, node to go NotReady, and pods to become un-evictable. If your K8S uses NFS storage, invest in NFS high availability, and keep critical system paths off NFS.
