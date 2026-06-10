---
title: "systemd Unit Timeout — When Services Refused to Start and the Server Went Dark"
date: 2026-06-11
weight: 100500
slug: "systemd-unit-timeout-service-down"
tags: ["linux", "systemd", "system", "troubleshooting", "service"]
categories: ["Troubleshooting"]
description: "A systemd unit timeout incident — how an NFS hang during boot caused multiple critical services to fail startup within the default 90s TimeoutStartSec, leaving the production server partially offline"
keywords: "systemd unit timeout, systemd TimeoutStartSec, journalctl -u service, systemd dependencies, linux service startup troubleshooting"
draft: false
featured: true
cover:
  image: ""
  caption: "systemd Unit Timeout — Troubleshooting"
---

## Common Search Queries

- systemd service timeout after reboot
- systemctl status shows failed due to timeout
- NFS mount hangs boot systemd
- _netdev option missing fstab
- TimeoutStartExceeded systemd
- remote-fs.target dependency failed
- nginx fails to start after reboot systemd timeout
- systemd unit start request repeated too quickly
- journalctl -u service timeout
- systemd dependency failed services

## The Incident

### Environment

- **OS**: Ubuntu 22.04 LTS
- **PostgreSQL**: 15 (running on local SSD)
- **Nginx**: 1.24 (serving reverse proxy, logging to NFS mount)
- **Tomcat**: 9 (deploying applications from NFS share)
- **NFS Mount**: Shared storage for Nginx access logs and Tomcat deployment artifacts, served by a dedicated NAS appliance
- **Scheduled Reboot**: Kernel update requiring restart, scheduled for 02:00 AM

### Timeline

At 02:00, the server initiated a graceful shutdown as scheduled after the kernel update. The hardware POST completed successfully, and GRUB loaded the new kernel. What followed, however, was a chain of failures that left the production server partially blind for 15 minutes.

### Symptoms

When the on-call engineer first logged in after the automated health check alerted:

- **SSH**: Working normally — the server was responsive
- **PostgreSQL**: Running and accepting connections — local data was unaffected
- **Nginx**: Returning HTTP 502 Bad Gateway — no upstream available
- **Tomcat**: Not responding on port 8080 or 8443 — completely unreachable
- **System monitoring**: Grafana dashboards showed gaps for Nginx and Tomcat metrics

At first glance, the server seemed healthy. SSH worked. PostgreSQL was fine. But the two services that carried user-facing traffic were both down. This is the classic "server is up but not really up" scenario that makes systemd timeout failures particularly dangerous.

## Background

### systemd Boot Sequence

Modern Linux systems using systemd follow a structured boot process. After the kernel initializes hardware, systemd (PID 1) starts the initial target (default.target, usually aliased to graphical.target or multi-user.target). Systemd resolves dependencies through a directed acyclic graph of **units** — services (.service), mount points (.mount), devices (.device), sockets (.socket), and targets (.target).

The boot order relevant to our incident:

1. **local-fs.target** — local filesystems are mounted
2. **remote-fs.target** — network filesystems (NFS, CIFS) are mounted
3. **network.target** — basic networking is available
4. **multi-user.target** — all services started for normal operation

Critical services like Nginx and Tomcat typically declare `After=remote-fs.target` when they depend on network-mounted storage. This means they will not start until remote-fs.target completes.

### Service Startup Timeout Mechanism

Every systemd service unit has a `TimeoutStartSec` directive. Its default value is **90 seconds** for most distributions, including Ubuntu 22.04. This parameter controls how long systemd waits for a service to report that it has started successfully.

But the timeout is not limited to service units. Mount units (.mount) also inherit timeout behavior. When a mount unit hangs (e.g., an NFS server is unreachable), systemd waits for the duration of `TimeoutStartSec`, then marks the mount as failed.

Key directives:

- **TimeoutStartSec=90** — default service start timeout
- **TimeoutStopSec=90** — default service stop timeout
- **DefaultTimeoutStartSec=90** — systemd compile-time default
- **TimeoutStartSec=infinity** — disable timeout (not recommended for production)

### systemd Dependency Graph

Understanding how units relate is crucial:

- **`After=`** — ordering only; this unit starts after the specified unit, but failure does not prevent startup
- **`Requires=`** — strong dependency; if the specified unit fails, this unit is stopped or deactivated
- **`Wants=`** — weak dependency; systemd attempts to start the specified unit, but failure is tolerated
- **`PartOf=`** — if the specified unit is stopped or restarted, this unit follows

In our case, Nginx's unit file (or its drop-in) likely had `After=remote-fs.target` and possibly `Requires=remote-fs.target`. When the NFS mount within remote-fs.target timed out, the entire target failed, cascading to dependent services.

### fstab Mount Options for systemd

The `/etc/fstab` file is parsed by systemd-fstab-generator at boot, which creates corresponding `.mount` units. The mount options directly affect systemd behavior:

| Option | Effect |
|--------|--------|
| `_netdev` | Marks the mount as a network filesystem; systemd waits for network to be available before attempting the mount |
| `nofail` | If the mount fails, systemd proceeds without marking it as failed — critical for non-essential network mounts |
| `x-systemd.automount` | Creates an automount unit instead of a mount unit; the filesystem is mounted on first access rather than at boot |
| `x-systemd.device-timeout=30s` | Timeout for the device to appear before systemd gives up |
| `defaults` | No special systemd behavior; mount is treated as a local filesystem dependency |

The absence of `_netdev` was the root cause. Without it, systemd treats the NFS mount as a local filesystem dependency, waiting for it during the boot sequence as part of `local-fs.target` or as an early dependency.

## Investigation

The following steps document the full investigation path. Each command reveals a piece of the puzzle.

### Step 1: Check Overall System State

```bash
systemctl list-units --failed
```

Output:
```
UNIT                        LOAD   ACTIVE SUB    DESCRIPTION
var-log-nginx.mount         loaded failed failed /var/log/nginx
nginx.service               loaded failed failed Nginx Web Server
tomcat.service               loaded failed failed Apache Tomcat
remote-fs.target            loaded failed failed Remote File Systems
```

Four failed units. The pattern was immediately suspicious — three services and one mount point were all in failed state. The mount `var-log-nginx.mount` pointed to the NFS share.

### Step 2: Check Individual Service Status

```bash
systemctl status nginx
```

```
● nginx.service - Nginx Web Server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: failed (Result: timeout) since Thu 2026-06-11 02:03:30 UTC; 12min ago
       Docs: man:nginx(8)
    Process: 892 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 893 ExecStart=/usr/sbin/nginx -g daemon on; (code=exited, status=0/SUCCESS)
   Main PID: 893 (code=exited, status=0/SUCCESS)
        CPU: 23ms

Jun 11 02:01:59 prod-web systemd[1]: Starting Nginx Web Server...
Jun 11 02:03:30 systemd[1]: nginx.service: Start request repeated too quickly.
Jun 11 02:03:30 systemd[1]: nginx.service: Failed with result 'timeout'.
Jun 11 02:03:30 systemd[1]: Failed to start Nginx Web Server.
```

Key observation: Nginx's ExecStartPre (config test) and ExecStart both exited with code 0 (success). Yet systemd reported timeout. This is unusual — normally timeout means the main process did not signal readiness within the window. But here the process started and exited successfully. The "start request repeated too quickly" message hinted at a dependency issue: systemd was retrying Nginx because a dependency (the NFS mount) eventually failed, and systemd was repeatedly trying to satisfy the dependency chain.

```bash
systemctl status tomcat
```

```
● tomcat.service - Apache Tomcat
     Loaded: loaded (/etc/systemd/system/tomcat.service; enabled; vendor preset: enabled)
     Active: failed (Result: dependency) since Thu 2026-06-11 02:01:59 UTC; 14min ago
       Docs: https://tomcat.apache.org
    Process: 845 ExecStart=/opt/tomcat/bin/startup.sh (code=exited, status=0/SUCCESS)
   Main PID: 845 (code=exited, status=0/SUCCESS)

Jun 11 02:01:30 prod-web systemd[1]: Starting Apache Tomcat...
Jun 11 02:01:59 prod-web systemd[1]: tomcat.service: Dependency failed, entering failed status.
Jun 11 02:01:59 prod-web systemd[1]: tomcat.service: Failed with result 'dependency'.
Jun 11 02:01:59 prod-web systemd[1]: Failed to start Apache Tomcat.
```

The "dependency" result was the critical clue. Tomcat did not fail on its own — something it depended on had failed first.

### Step 3: Check the Boot Journal

```bash
journalctl -b -1 | grep -i "timeout\|failed\|error" | tail -30
```

```
Jun 11 02:00:45 prod-web kernel: NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:00:45 prod-web kernel: NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:01:05 prod-web systemd[1]: var-log-nginx.mount: Mount process timed out
Jun 11 02:01:05 prod-web systemd[1]: var-log-nginx.mount: Failed with result 'timeout'
Jun 11 02:01:05 prod-web systemd[1]: Failed to mount /var/log/nginx
Jun 11 02:01:59 prod-web systemd[1]: remote-fs.target: Found dependency on var-log-nginx.mount
Jun 11 02:01:59 prod-web systemd[1]: remote-fs.target: Job verilog-nginx.mount/start failed with result 'timeout'
Jun 11 02:01:59 prod-web systemd[1]: remote-fs.target: Triggering dependency failures
Jun 11 02:03:30 prod-web systemd[1]: nginx.service: Start request repeated too quickly
Jun 11 02:03:30 prod-web systemd[1]: nginx.service: Failed with result 'timeout'
Jun 11 02:03:30 prod-web systemd[1]: Failed to start Nginx Web Server
```

The NFS kernel module reported error -512 (which is `ERESTARTSYS`, a kernel-internal restart indicator, but in NFS context it typically means the server is unreachable). Then the mount unit timed out after what appeared to be 20 seconds (02:00:45 to 02:01:05), though this was the timeout for the mount command itself, not systemd's full TimeoutStartSec. systemd then spent additional time retrying before failing the target and its dependent services.

### Step 4: Check Unit Dependencies

```bash
systemctl list-dependencies nginx
```

```
nginx.service
● ├─system.slice
● ├─nginx.service
● └─remote-fs.target
●   └─var-log-nginx.mount
```

This confirmed the dependency chain. Nginx depended on `remote-fs.target`, which depended on `var-log-nginx.mount`, which was the NFS mount that failed.

```bash
systemctl list-dependencies tomcat
```

```
tomcat.service
● ├─system.slice
● ├─tomcat.service
● └─remote-fs.target
●   └─var-log-nginx.mount
```

Tomcat had the same dependency chain.

### Step 5: Investigate NFS Mounts

```bash
mount | grep nfs
```

No output. The NFS mount was not active.

```bash
systemctl status remote-fs.target
```

```
● remote-fs.target - Remote File Systems
     Loaded: loaded (/usr/lib/systemd/system/remote-fs.target; static)
     Active: failed (Result: dependency) since Thu 2026-06-11 02:01:59 UTC; 15min ago
       Docs: man:systemd.special(7)
```

```bash
journalctl -u remote-fs.target
```

```
Jun 11 02:00:40 prod-web systemd[1]: Reached target Remote File Systems.
Jun 11 02:00:45 prod-web systemd[1]: var-log-nginx.mount: Mount process timed out
Jun 11 02:00:45 prod-web systemd[1]: remote-fs.target: Job var-log-nginx.mount/start failed with result 'timeout'
Jun 11 02:00:45 prod-web systemd[1]: remote-fs.target: Triggering dependency failures
Jun 11 02:00:45 prod-web systemd[1]: remote-fs.target: Dependency failed, entering failed status
```

The journal showed that remote-fs.target was initially reached but then immediately failed when the NFS mount timed out.

### Step 6: Check /etc/fstab

```bash
cat /etc/fstab | grep nfs
```

```
nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0
```

Found the problem. The NFS entry used `defaults` instead of `_netdev`. This is the smoking gun.

### Step 7: Check NFS Server Connectivity

```bash
timeout 5 showmount -e nfs-server
```

```
showmount: RPC: Program not registered
```

The NFS server was not responding. During maintenance, the NAS appliance had been taken offline, which was the root trigger.

### Step 8: Check System Log for Boot-time NFS Errors

```bash
grep -i "nfs\|mount" /var/log/syslog | grep -i "error\|fail\|timeout"
```

```
Jun 11 02:00:41 prod-web kernel: [   15.432] NFS: Registering the id_resolver key type
Jun 11 02:00:41 prod-web kernel: [   15.432] FS-Cache: Netfs 'nfs' registered for caching
Jun 11 02:00:42 prod-web kernel: [   16.105] NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:00:42 prod-web kernel: [   16.105] NFS: nfs4_discover_server_trunking unhandled error -512
Jun 11 02:00:45 prod-web systemd[1]: var-log-nginx.mount: Mount process timed out
```

The kernel's NFS client attempted to discover the server trunking capability, but the server was unreachable, returning error -512 repeatedly until the mount timeout was reached.

## Root Cause

The root cause chain is straightforward:

### Primary Cause: Missing _netdev in fstab

The `/etc/fstab` entry for the NFS share was:

```
nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0
```

The `defaults` option does not include `_netdev`. When systemd-fstab-generator processes this entry, it does not recognize it as a network filesystem. Without `_netdev`, the mount unit is treated as a local mount dependency and is started during early boot, before the network is fully ready.

### Secondary Cause: NFS Server Unavailable

The NAS appliance providing the NFS share was down for scheduled maintenance. The maintenance window overlapped with the server reboot, creating a perfect storm.

### How the Failure Propagated

1. **Boot starts** — systemd begins processing mount units
2. **NFS mount attempt** — `var-log-nginx.mount` tries to mount `nfs-server:/exports/logs`
3. **Mount hangs** — the NFS server is unreachable; the mount command blocks
4. **Timeout expires** — after `TimeoutStartSec` (default 90s, though the mount command itself timed out earlier), systemd marks `var-log-nginx.mount` as failed
5. **Target failure** — `remote-fs.target` depends on `var-log-nginx.mount`; it enters failed state
6. **Cascading failure** — Nginx and Tomcat both have `After=remote-fs.target` and/or `Requires=remote-fs.target`; they cannot start
7. **systemd retries** — systemd attempts to restart Nginx, but the dependency chain is broken; each attempt fails
8. **Start request repeated too quickly** — systemd detects repeated rapid failures and stops trying

### Why PostgreSQL Was Unaffected

PostgreSQL stored its data on local SSD storage. Its systemd unit file had no dependency on `remote-fs.target` or any NFS mount. The database service started independently and was fully operational throughout the incident.

### The Partial Outage Illusion

The server appeared healthy because:
- SSH daemon (sshd) has no dependency on remote-fs.target
- PostgreSQL has no NFS dependency
- Basic system services (syslog, cron, etc.) run from local storage
- The kernel booted successfully and networking was functional

Only services that depended on the NFS mount were affected. This selective failure makes this class of incident harder to detect with simple "is the server up?" checks.

## Resolution

### Emergency Fix

The immediate priority was to restore Nginx and Tomcat. The fix required correcting the fstab entry and re-mounting.

**Step 1: Fix /etc/fstab**

Edit `/etc/fstab` and update the NFS mount entry:

```
# Old (broken):
# nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0

# New (fixed):
nfs-server:/exports/logs  /var/log/nginx  nfs  _netdev,noexec,nofail,x-systemd.automount,x-systemd.device-timeout=30s  0  0
```

Key option breakdown:

| Option | Purpose |
|--------|---------|
| `_netdev` | Tells systemd this is a network filesystem; waits for network readiness |
| `nofail` | If mount fails, systemd proceeds without marking as failed |
| `x-systemd.automount` | Mount on first access instead of at boot — critical for network filesystems |
| `x-systemd.device-timeout=30s` | Give up waiting for the NFS server after 30 seconds |
| `noexec` | Security hardening — prevents execution from mounted filesystem |

**Step 2: Reload systemd**

```bash
systemctl daemon-reload
```

This tells systemd to re-read the unit files generated from fstab.

**Step 3: Restart Failed Services**

```bash
systemctl restart nginx tomcat
```

**Step 4: Verify Service Status**

```bash
systemctl is-active nginx tomcat
```

Expected output:
```
active
active
```

**Step 5: Verify Mount Status**

```bash
mount | grep nfs
```

With `x-systemd.automount`, the mount may not appear immediately. It will be mounted on first access. To force the mount:

```bash
systemctl restart var-log-nginx.mount
```

Or simply access the mount point:
```bash
ls -la /var/log/nginx/
```

### Long-term Preventive Measures

#### 1. Add nofail to All Network Filesystem Mounts

Audit all entries in `/etc/fstab` that reference network filesystems (NFS, CIFS, GlusterFS, etc.). Every network mount should include `nofail` to prevent a mount failure from blocking the boot sequence.

```bash
grep -E "nfs|cifs|smb|gluster|fuse" /etc/fstab
```

For each entry, add `_netdev,nofail,x-systemd.automount` to the options field.

#### 2. Implement x-systemd.automount for NFS

The `x-systemd.automount` option creates a systemd automount unit instead of a mount unit. The filesystem is mounted on-demand when a process first accesses the mount point. This has several advantages:

- Boot is not blocked by NFS availability
- If the NFS server is down, services can still start (they will encounter an I/O error only when actually accessing the mount)
- The mount is retried automatically when accessed
- Systemd can reload the mount if the network becomes available later

#### 3. Review Critical Service Dependencies

For each critical service, review its systemd unit file for unnecessary filesystem dependencies:

```bash
systemctl cat nginx
```

Look for `After=`, `Requires=`, and `Wants=` directives. Consider whether all listed dependencies are truly necessary for the service to function. If a dependency is "nice to have" rather than "essential", downgrade `Requires=` to `Wants=` and add `nofail` to the related mount.

#### 4. Add Boot-time Monitoring

The best defense against silent partial failures is monitoring. Add these checks:

**Prometheus blackbox-style monitoring** for boot time:
```yaml
# prometheus rule
- alert: SystemdFailedUnits
  expr: node_systemd_unit_state{state="failed"} > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Systemd failed units detected on {{ $labels.instance }}"
```

**Custom boot health check script**:
```bash
#!/bin/bash
# /usr/local/bin/boot-health-check.sh
# Run as a systemd oneshot service after multi-user.target

FAILED_UNITS=$(systemctl list-units --failed --no-legend --no-pager | wc -l)
if [ "$FAILED_UNITS" -gt 0 ]; then
    systemctl list-units --failed --no-legend --no-pager | \
        mail -s "Boot Health Check Failed on $(hostname)" ops@example.com
fi
```

**Grafana dashboard alert** — monitor the `node_systemd_unit_state` metric and trigger alerts on any failed unit.

#### 5. Adjust TimeoutStartSec for Services That Genuinely Need More Time

Some services require more than 90 seconds to start. For those, set an appropriate `TimeoutStartSec` in a drop-in configuration rather than globally:

```bash
mkdir -p /etc/systemd/system/nginx.service.d
cat > /etc/systemd/system/nginx.service.d/timeout.conf << 'EOF'
[Service]
TimeoutStartSec=300s
EOF
systemctl daemon-reload
```

Note: This was not the fix for this incident. Increasing the timeout without fixing the fstab would only delay the failure, not prevent it.

#### 6. Test Reboot Procedure in Staging

Before any future kernel update or reboot:

1. Execute a staged reboot in the testing environment
2. Automatically validate all critical services post-boot:
   ```bash
   # post-reboot validation script
   for svc in nginx tomcat postgresql sshd; do
       if ! systemctl is-active --quiet "$svc"; then
           echo "FAIL: $svc is not active"
           exit 1
       fi
   done
   echo "All critical services are running"
   ```
3. Simulate NFS server failure during boot to validate that `nofail` and `x-systemd.automount` work as expected

#### 7. Document Systemd Unit Dependency Architecture

Create a living document that maps every critical service to its systemd dependencies. This helps engineers understand the blast radius when a dependency fails:

```bash
# Generate dependency graph for all critical services
for unit in nginx tomcat postgresql sshd; do
    echo "=== $unit ==="
    systemctl list-dependencies "$unit" --no-legend --no-pager
done
```

## Lessons Learned

### 1. The fstab `defaults` Trap

The `defaults` mount option in `/etc/fstab` is deceptively named. It does not mean "safe defaults for production" — it means the kernel's default set of mount flags (rw, suid, dev, exec, auto, nouser, async). Crucially, it does not include `_netdev`, `nofail`, or any of the systemd-specific options that protect against network filesystem failures.

**Lesson**: Never use plain `defaults` for network filesystem mounts. Always explicitly include `_netdev` and `nofail`.

### 2. Partial Outages Are Harder to Detect

A server that is "up" (SSH reachable, kernel running) but missing critical services is more dangerous than a fully down server. Monitoring must check service health, not just server reachability.

**Lesson**: Layer your monitoring — check port availability, process health, and application-level responses independently.

### 3. Timeout Does Not Mean the Service Is Slow

When systemd reports a timeout, the instinct is to increase TimeoutStartSec. But a timeout often means a dependency is broken, not that the service needs more time. Always investigate the dependency chain first.

**Lesson**: When you see a timeout, ask "what is it waiting for?" before adjusting timing parameters.

### 4. systemd Dependency Chains Create Blast Radius

One failed mount unit can take down multiple unrelated services. The dependency graph of systemd units must be understood and audited regularly. A service that does not truly need a remote filesystem should not declare a dependency on it.

**Lesson**: Minimize dependency chains. Prefer `Wants=` over `Requires=` for non-essential dependencies. Use `nofail` wherever possible.

### 5. Scheduled Maintenance Windows Need Coordination

The NFS server maintenance and the server reboot were both scheduled but not coordinated. A simple cross-team communication would have prevented this incident.

**Lesson**: When multiple maintenance operations are scheduled, verify that the dependencies between systems are accounted for.

### 6. Automount Is Your Friend

`x-systemd.automount` is not just a convenience feature — it is a reliability mechanism. By deferring the mount to first access, you decouple service startup from filesystem availability. This makes the system more resilient to transient network issues.

**Lesson**: Use automount for all network filesystems unless there is a specific reason to mount at boot time.

### 7. Always Have a Boot Validation Procedure

A simple post-boot health check that verifies all critical services are running would have caught this incident immediately. Without it, the 15-minute outage was discovered only by an unrelated automated health check.

**Lesson**: Implement boot-time validation. A 5-line script is infinitely better than a 15-minute outage.

## Summary

### Incident Timeline

| Time (UTC) | Event |
|------------|-------|
| 02:00 | Server initiates scheduled reboot for kernel update |
| 02:00:40 | kernel boots, systemd initializes targets |
| 02:00:42 | systemd attempts NFS mount for /var/log/nginx |
| 02:00:45 | NFS mount times out (NFS server unreachable) |
| 02:01:05 | var-log-nginx.mount marked as failed |
| 02:01:05 | remote-fs.target enters failed state due to dependency |
| 02:01:30 | Tomcat starts, immediately fails due to dependency failure |
| 02:01:59 | Tomcat marked as failed (result: dependency) |
| 02:03:30 | Nginx retries exhausted, marked as failed (result: timeout) |
| 02:15 | Automated alert triggers on-call engineer |
| 02:17 | Engineer fixes fstab, reloads systemd, restarts services |
| 02:17:30 | Nginx and Tomcat confirmed active |

Total downtime for user-facing services: 15 minutes.

### fstab Configuration Comparison

**Before (broken):**
```
nfs-server:/exports/logs  /var/log/nginx  nfs  defaults  0  0
```

**After (fixed):**
```
nfs-server:/exports/logs  /var/log/nginx  nfs  _netdev,noexec,nofail,x-systemd.automount,x-systemd.device-timeout=30s  0  0
```

The fix adds four critical options: `_netdev` to classify the mount as network-based, `nofail` to prevent boot blocking, `x-systemd.automount` to defer mounting to first access, and `x-systemd.device-timeout=30s` to bound the wait time. Together, these options ensure that an unreachable NFS server will never again take down the production web services.
