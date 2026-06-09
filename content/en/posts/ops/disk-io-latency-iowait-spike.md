---
title: "Disk IO Latency Spike — When iowait Reached 90% and the Node Stopped Responding"
date: 2026-06-09
weight: 100440
slug: "disk-io-latency-iowait-spike"
tags: ["storage", "troubleshooting", "linux", "disk", "performance"]
categories: ["Troubleshooting"]
description: "A disk IO latency incident — how a malfunctioning SAS expander in the JBOD enclosure caused disk latency to spike from 2ms to 8000ms, bringing all IO-bound services to a halt"
keywords: "disk io latency, iowait high, linux disk troubleshooting, iostat, iotop, linux io bottleneck, disk latency spike"
draft: false
featured: true
cover:
  image: ""
  caption: "Disk IO Latency — Troubleshooting"
---

## Common Search Queries

If you landed here from a search engine, you are likely experiencing one of these symptoms:

- `iowait` in `top` shows 80-99%, server barely responsive
- `iostat -x` shows `await` in the thousands of milliseconds
- PostgreSQL / MySQL queries that normally take 10ms now take 30+ seconds
- Commands like `ls` or `ssh` hang for tens of seconds
- `dmesg` filled with `I/O error` or `SCSI` related messages
- Processes stuck in `D` state (uninterruptible sleep)
- Alert: "node disk latency > 5s" from your monitoring system

This article walks through a real production incident where a faulty SAS expander cable caused disk latency to spike from 2ms to over 8000ms, taking down a PostgreSQL database cluster and five microservices.

---

## The Incident

### Environment

| Component | Specification |
|---|---|
| Server | Dell PowerEdge R750 |
| Storage | JBOD enclosure with 12x SATA SSD |
| Filesystem | ext4 on LVM |
| Database | PostgreSQL 15 |
| Applications | 5 microservices (Java/Go/Python) |
| OS | Ubuntu 22.04 LTS, Kernel 5.15 |

### Timeline

**14:23** — Application monitoring dashboards light up. P99 API latency jumps from 50ms to over 5 seconds. PostgreSQL query times go from 10ms to 30s+.

**14:24** — On-call engineer receives alerts:
- `node_disk_io_time_seconds_total` showing sustained high IO time
- "Node disk latency > 5s" threshold breached
- PostgreSQL connection pool nearly exhausted

**14:25** — SSH into the node succeeds but each keystroke takes 3-5 seconds to echo. Commands take 30-60 seconds to complete.

**14:26** — `top` shows `iowait` at 93%. Load average at 85 (baseline: 4-8).

### Symptoms at a Glance

- API latency: 100x normal
- `iowait`: 90%+
- SSH: typing lags, interactive sessions nearly unusable
- PostgreSQL: queries timing out, connection pool filling up
- Microservices: health check probes failing, instances being removed from service discovery
- System logs: unable to write, `syslog-ng` blocked
- New relic / Prometheus: disk latency metric well above 5s threshold

---

## Background

Before diving into the investigation, a quick refresher on the Linux IO stack and what `iowait` actually means.

### Linux IO Stack

When a process reads or writes a file, the request travels through several layers:

```
Application (PostgreSQL)
    ↓
VFS (Virtual File System)
    ↓
Filesystem (ext4)
    ↓
Block Layer (mq-deadline / kyber / none)
    ↓
SCSI Mid-layer (retries, timeout handling)
    ↓
SCSI Low-level Driver (HBA/controller)
    ↓
Physical Device (SAS expander → SATA SSD)
```

Each layer adds its own queuing, merging, and timeout logic. A failure at the physical layer (e.g., a bad SAS cable) propagates all the way up as slow or failed IO.

### What is iowait?

`iowait` in `top` / `mpstat` represents the percentage of time the CPU is idle while waiting for at least one IO operation to complete.

Key points:
- **iowait is CPU-idle time**, not CPU-busy time. The CPU has nothing to do because it is waiting for IO.
- High iowait means the **storage subsystem is the bottleneck**, not the CPU.
- iowait does NOT tell you which disk is slow, which process is doing IO, or what type of IO (read vs write). It is merely a signal that something is wrong with the IO path.

### IOPS vs Latency

| Metric | What it measures | When to care |
|---|---|---|
| IOPS | Number of IO operations per second | Throughput-bound workloads |
| Latency (await) | Time per IO operation from submission to completion | Latency-sensitive workloads (DB, real-time) |
| %util | Percentage of time the device was busy serving requests | Does not indicate saturation on modern NVMe/SSD |

In this incident, IOPS dropped because each request took 8000ms+ — but the key metric was **latency**, not IOPS.

---

## Investigation

### Step 1: Check System Load

```bash
# First thing: see what top tells us
top
```

Output:
```
top - 14:26:18 up 42 days,  3:11,  2 users,  load average: 85.32, 42.18, 18.07
Tasks: 345 total,   1 running, 284 sleeping,  60 stopped,   0 zombie
%Cpu(s):  2.1 us,  1.8 sy,  0.0 ni, 93.7 id, 93.1 wa,  0.0 hi,  0.3 si,  0.0 st
MiB Mem : 128898.6 total,   1245.3 free,  89234.5 used,  38418.8 buff/cache
MiB Swap:   2048.0 total,      0.0 free,   2048.0 used.  38894.2 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1234 postgres  20   0  ......
```

Key findings:
- `iowait` (wa): **93.1%** — critical
- Load average: **85.32** — baseline was 4-8, now 10x higher
- System is still alive but barely responsive

```bash
uptime
# 14:26:18 up 42 days, 3:11, 2 users, load average: 85.32, 42.18, 18.07
```

### Step 2: Identify Disk Latency

```bash
# Extended disk statistics, 1-second intervals, 5 samples
iostat -x 1 5
```

Output:
```
Device     r/s     w/s    rkB/s    wkB/s  await  svctm  %util
sda       12.3     8.7    456.2    234.1   2.5    0.8    1.7
sdb        0.4     0.2     12.3      4.1 8342.1 1230.5 75.3
sdc        8.2     5.1    321.4    123.7   3.1    0.9    1.2
```

Key findings:
- **sdb**: `await` of **8342ms** (normal: 2-5ms for SSD)
- **sdb**: `%util` of **75.3%** — but more importantly, the latency is astronomical
- Other disks (sda, sdc) are fine, suggesting this is a **per-disk** issue, not a controller-wide issue

The command `iostat -x 1 5` deserves explanation:
- `-x`: extended statistics (includes `await`, `svctm`, `%util`)
- `1 5`: refresh every 1 second, 5 times total

Key columns to watch:
- **await**: average IO response time (includes queuing + service time). This is your best latency indicator.
- **svctm**: average service time (actual IO processing time, excluding queue). On modern SSDs this should be <1ms.
- **%util**: percentage of time the device was busy. A single slow disk can show 100% even with low IOPS.
- **r_await / w_await**: read/write-specific await on newer kernels.

{{< alert info >}}
On modern kernels, `svctm` is deprecated and may show 0. Use `await` + `r_await` / `w_await` instead.
{{< /alert >}}

### Step 3: Identify IO-Heavy Processes

```bash
# Show only processes currently doing IO
iotop -o
```

Output:
```
TOTAL DISK READ: 0.4 K/s | TOTAL DISK WRITE: 0.2 K/s
  PID  PRIO  DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 1234 be/4    0.0 K/s    0.1 K/s  0.00 % 99.99 %  postgres: writer
 5678 be/4    0.0 K/s    0.0 K/s  0.00 % 99.99 %  syslog-ng
 9012 be/4    0.0 K/s    0.1 K/s  0.00 % 99.99 %  postgres: wal-writer
```

Key findings:
- **PostgreSQL writer and WAL writer**: stuck at 99.99% IO wait
- **syslog-ng**: also stuck — the system cannot even write logs
- Almost no actual throughput (0.4 K/s read, 0.2 K/s write) — the disk is effectively hung

{{< alert warning >}}
When syslog-ng itself is blocked on IO, you lose the ability to log the incident. In production, consider a separate logging disk or remote logging.
{{< /alert >}}

### Step 4: Check SCSI Layer

The SCSI mid-layer is often the key to understanding disk IO issues because it handles command retries and timeouts.

```bash
# Check device state
cat /sys/block/sdb/device/state
# → "running"

# Check SCSI timeout value (default is 30s)
cat /sys/block/sdb/device/timeout
# → 30
```

The device state is "running" — meaning the kernel still thinks the device is operational. The timeout is 30 seconds, which means the SCSI layer will wait 30s before declaring a command failed.

```bash
# Check kernel messages for SCSI errors
dmesg | grep -i "scsi\|sdb\|I/O error" | tail -20
```

Expected output for this scenario:
```
[8372912.451] sd 0:0:1:0: task abort: scsi 0x00000000 failed to finish within 30s
[8372912.452] sd 0:0:1:0: attempting task abort! scsi(0x00000000)
[8372942.512] sd 0:0:1:0: task abort: scsi 0x00000001 failed to finish within 30s
[8372942.513] sd 0:0:1:0: attempting task abort! scsi(0x00000001)
[8372972.573] sd 0:0:1:0: task abort: scsi 0x00000002 failed to finish within 30s
[8372972.574] sd 0:0:1:0: attempting task abort! scsi(0x00000002)
[8373002.634] sd 0:0:1:0: task abort: scsi 0x00000003 failed to finish within 30s
```

Every 30 seconds, a new SCSI command times out and the mid-layer retries. This pattern confirms the issue is at the **SCSI/physical layer**, not the filesystem or application.

```bash
# Also check for SCSI error counters (not available on all kernels)
cat /sys/block/sdb/device/ioerr_cnt 2>/dev/null
# May not exist — depends on kernel version and driver
```

### Step 5: Check Disk SMART Data

```bash
# SMART data for the affected disk
smartctl -a /dev/sdb | grep -i "reallocated\|pending\|uncorrect\|error"
```

Expected output when the disk itself is healthy but the connection is bad:
```
SMART overall-health SELF-ASSESSMENT RESULT: PASSED
  5 Reallocated_Sector_Ct: 0
196 Reallocated_Event_Count: 0
197 Current_Pending_Sector: 0
198 Offline_Uncorrectable: 0
```

**SMART data looks clean** — the disk itself is healthy. This rules out a failing SSD and points firmly at the **connection/controller/enclosure**.

{{< alert tip >}}
Clean SMART data + SCSI timeouts = suspect the connection (cable, expander, backplane, HBA). Dirty SMART data + SCSI timeouts = suspect the disk itself.
{{< /alert >}}

### Step 6: Check Filesystem

```bash
# Check filesystem-level errors
dmesg | grep -i "ext4\|xfs\|journal" | tail -10
```

Expected output:
```
[8372912.455] EXT4-fs warning (device dm-0): ext4_end_bio:340: I/O error writing to inode 1234568
[8372972.576] EXT4-fs warning (device dm-0): ext4_end_bio:340: I/O error writing to inode 1234592
```

The filesystem is reporting IO errors, but these are **symptoms** of the underlying SCSI issue, not the root cause. The filesystem itself (ext4 journal, superblock) is intact.

### Step 7: Trace Block Layer Latency

```bash
# Read block device statistics directly
cat /sys/block/sdb/stat
```

Output format (fields from Documentation/block/stat.txt):
```
   read I/Os      merge  sectors   ticks     write I/Os     merge  sectors   ticks     in_flight  io_ticks  time_in_queue
   2847291        1245   88928345  23948231  8391234        2341   67123456  928371233  127        89234723  123781237
```

The key value is `io_ticks` — the total time (in milliseconds) the device has been busy. If this is growing rapidly while `in_flight` is non-zero, the device is stuck.

In our case, `io_ticks` for sdb was growing by ~1000ms per second — meaning the device was 100% busy and every IO took at least 1 second.

```bash
# Also check per-disk latency histograms (if CONFIG_LATENCYTOP is enabled)
# This requires root and is distribution-dependent
# echo 1 > /proc/sys/kernel/latencytop  (if available)
```

---

## Root Cause

### The Smoking Gun: Faulty SAS Expander Cable

After removing the affected disk from use and inspecting the hardware, the Dell R750's JBOD enclosure was found to have a **SAS expander cable with intermittent connection**. The cable had been partially dislodged (possibly during a recent rack maintenance) and was making poor contact.

### The Failure Cascade

Here is the full chain of events:

```
SAS cable intermittent contact
    ↓
SAS expander loses link with disk intermittently
    ↓
SCSI commands to the disk time out (30s default timeout)
    ↓
SCSI mid-layer retries (up to 5 retries by default)
    ↓
Each IO request is blocked for 30-150 seconds
    ↓
PostgreSQL WAL flush hangs → all writes block
    ↓
PostgreSQL writer processes enter D state (uninterruptible sleep)
    ↓
syslog-ng blocks trying to write audit logs
    ↓
All IO-bound processes accumulate in D state
    ↓
System load skyrockets (85+ load average)
    ↓
Node becomes nearly unresponsive
```

### Why 30-150 Seconds Per IO?

The SCSI mid-layer operates with these defaults:

| Parameter | Default | Effect |
|---|---|---|
| SCSI timeout | 30 seconds | Wait time before declaring a command failed |
| Max retries | 5 | Number of times to retry before giving up |
| Total worst case | 30s × 5 = 150s | Maximum time one IO request can be blocked |

So a single `write()` call from PostgreSQL could block for up to 150 seconds before returning an error. During those 150 seconds, the entire database is effectively frozen.

### D State Explained

Processes waiting for IO are in **D state** (uninterruptible sleep, TASK_UNINTERRUPTIBLE). Unlike S state (interruptible sleep), D state processes **cannot be killed** — not even with `SIGKILL`. They will only exit D state when the IO operation completes or the device is taken offline.

This is why the node had 60+ D-stated processes and why `reboot` was being considered as a last resort.

---

## Resolution

### Emergency Steps

{{< alert danger >}}
Read carefully before executing any commands. Taking a disk offline that contains active data can cause data loss. Ensure you have identified the correct disk.
{{< /alert >}}

#### 1. Identify the Affected Disk

```bash
# Confirm which disk is bad
smartctl -a /dev/sdb | grep -i "reallocated\|pending\|uncorrect"
dmesg | grep -i "sdb\|I/O error" | tail -10
```

#### 2. Check Filesystem Layout

```bash
# Find which LVM VG/LV and mount point the bad disk belongs to
lsblk /dev/sdb
# sdb           8:16   0   3.7T  0 disk
# └─vg_data-lv_data 253:0 0 3.7T 0 lvm  /var/lib/postgresql

# If this is a critical volume, check if you have replicas or can failover
```

#### 3. Attempt SCSI Bus Rescan

Sometimes the kernel can recover the device with a bus rescan:

```bash
# Rescan SCSI bus (host0 = first HBA, change as needed)
echo "- - -" > /sys/class/scsi_host/host0/scan
```

If the cable issue is intermittent, the rescan might re-establish the connection. In our case, it did not — the cable was too far gone.

#### 4. Take the Disk Offline

If rescan does not work, take the disk offline:

```bash
# Take disk offline (immediately fails all pending IO)
echo offline > /sys/block/sdb/device/state
```

When you set a device to `offline`, the SCSI layer immediately fails all pending commands. This causes:
- All D-stated processes to be released (they receive IO errors)
- PostgreSQL to promote a replica or enter crash recovery
- The system to become responsive again

{{< alert danger >}}
Setting a disk to `offline` will cause filesystem errors on affected filesystems. You must remount or fsck before use. In our case, the PostgreSQL data was on a separate LV that could be failed over to a replica.
{{< /alert >}}

#### 5. Remount Filesystem (if needed)

If the filesystem is corrupt or in an inconsistent state:

```bash
# Check and remount
umount /var/lib/postgresql
fsck -y /dev/vg_data/lv_data
mount /var/lib/postgresql
```

#### 6. Last Resort: Reboot

If the system is completely unresponsive and you cannot take disks offline:

```bash
# Sync and reboot
sync; reboot
```

In extreme cases, you may need a hard reset (power cycle). This should be the absolute last resort.

### Long-Term Fixes

#### 1. Replace Faulty Hardware

```bash
# After replacement, verify all disks are detected
lsscsi
# [0:0:0:0]    disk    Dell     PERC H755        5.11  /dev/sda
# [0:0:1:0]    disk    ATA      INTEL SSDSC2KB   XCV1  /dev/sdb
# [0:0:2:0]    disk    ATA      INTEL SSDSC2KB   XCV1  /dev/sdc
```

#### 2. Reduce SCSI Timeout for Faster Failover

```bash
# Reduce SCSI timeout from 30s to 10s for quicker detection
echo 10 > /sys/block/sdb/device/timeout

# Make persistent via udev rule
cat > /etc/udev/rules.d/99-scsi-timeout.rules << 'EOF'
ACTION=="add", SUBSYSTEM=="scsi", ATTR{device/type}=="0", ATTR{device/timeout}="10"
EOF
```

#### 3. Add Multipath IO for Critical Disks

```bash
# Install multipath tools
apt-get install multipath-tools

# Configure multipath
cat >> /etc/multipath.conf << 'EOF'
defaults {
    user_friendly_names yes
    polling_interval 5
    path_selector "round-robin 0"
    path_grouping_policy multibus
}

multipaths {
    multipath {
        wwid  36xxxx...  # Replace with actual disk WWID
        alias  mpath_data
    }
}
EOF

systemctl restart multipathd
```

#### 4. Set Up Monitoring

**Prometheus alert rules (PromQL):**

```yaml
# Disk latency alerts
- alert: DiskLatencyHigh
  expr: |
    rate(node_disk_io_time_seconds_total{device=~"sd[a-z]"}[5m])
    / rate(node_disk_reads_completed_total{device=~"sd[a-z]"}[5m])
    > 0.1
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "Disk {{ $labels.device }} average latency > 100ms"

- alert: DiskLatencyCritical
  expr: |
    rate(node_disk_io_time_seconds_total{device=~"sd[a-z]"}[5m])
    / rate(node_disk_reads_completed_total{device=~"sd[a-z]"}[5m])
    > 1.0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Disk {{ $labels.device }} average latency > 1s"

# SCSI error detection (requires textfile collector or custom exporter)
- alert: SCSIErrorsDetected
  expr: |
    node_scsi_errors_total > 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "SCSI errors detected on {{ $labels.device }}"
```

**Production monitoring checklist:**

```bash
# Continuous monitoring with iostat
iostat -x 60 > /var/log/iostat.log &

# Monitor SCSI errors from dmesg
dmesg --follow | grep --line-buffered "I/O error" | while read line; do
    logger -p user.warning "SCSI_ERROR: $line"
done
```

---

## Lessons Learned

### What Went Well

1. **Monitoring caught it early**: Prometheus latency alerts fired within 1 minute of the incident
2. **Read-only filesystem saved us**: Some filesystems remounted as read-only under the IO storm, preventing further corruption
3. **Replica was available**: PostgreSQL replica took over after the primary was taken offline

### What Went Wrong

1. **No SCSI error monitoring**: We had disk latency alerts but no monitoring of SCSI errors or dmesg patterns
2. **Single path to storage**: No multipath IO. A single cable failure should not bring down the entire storage stack
3. **syslog-ng on same disk**: The logging daemon competed with the database for IO, and when the disk failed, we lost all logs from the incident period
4. **Default SCSI timeout too long**: 30 seconds is designed for spinning rust. For SSD-based systems with fast failover, 10 seconds or less is more appropriate
5. **No IO latency SLO**: We had CPU and memory SLOs but no disk latency SLO. The alert "node disk latency > 5s" was added after this incident
6. **No automated disk fencing**: Taking the disk offline required manual SSH intervention — which was nearly impossible with 90% iowait

### Key Takeaways

- **iowait does not diagnose — it signals**. High iowait tells you IO is the bottleneck, but you still need `iostat`, `iotop`, and `dmesg` to find the cause.
- **SCSI layer is the canary**. SCSI timeouts almost always indicate a hardware or cabling issue, not a software problem.
- **D state is dangerous**. Processes in uninterruptible sleep cannot be killed and accumulate, making the system progressively worse.
- **Monitor the IO path, not just the disk**. Disk SMART data can show "PASSED" while the connection to the disk is broken.
- **Test your cable management**. A partially dislodged SAS cable during rack maintenance triggered this entire incident.

---

## Summary

### Incident Timeline

```
14:23 — App latency spikes 100x (50ms → 5s+)
14:24 — Alerts fire: node disk latency > 5s
14:25 — Engineer SSHs in (painfully slow)
14:26 — top shows 93% iowait, load average 85
14:30 — iostat identifies sdb with 8000ms await
14:35 — iotop shows postgres + syslog-ng stuck
14:40 — dmesg confirms SCSI timeouts every 30s
14:45 — SMART data shows clean disk — cable suspected
14:50 — Offline sdb, processes released
14:55 — PostgreSQL replica promoted, service restored
15:30 — Hardware inspection confirms loose SAS cable
16:00 — Cable reseated, disk verified, replica re-synced
Total impact: ~90 minutes of partial or complete service degradation
```

### Flow Diagram

```
                     ┌───────────────────────┐
                     │  SAS Cable Failure     │
                     │  (Intermittent Contact) │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  SAS Expander Link     │
                     │  Drops Intermittently  │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  SCSI Command Timeout  │
                     │  (30s default)         │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  SCSI Retry (x5 max)   │
                     │  Each IO blocked 30-150s│
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  Block Layer Stalls    │
                     │  mq-deadline queues fill│
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  ext4 Journal Freezes  │
                     │  No writes possible    │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  PostgreSQL WAL Blocked │
                     │  All writes D-state    │
                     └───────────┬───────────┘
                                 │
                     ┌───────────▼───────────┐
                     │  System Unresponsive   │
                     │  93% iowait, Load 85   │
                     └───────────────────────┘
```

### Commands Quick Reference

| Command | Purpose |
|---|---|
| `top` | Quick check: iowait, load average, D-state processes |
| `iostat -x 1 5` | Per-disk latency (`await`), utilization (`%util`) |
| `iotop -o` | Which processes are doing IO right now |
| `dmesg \| grep -i scsi` | SCSI errors and timeouts |
| `cat /sys/block/sdX/device/state` | Disk device state (running/offline) |
| `cat /sys/block/sdX/device/timeout` | SCSI timeout in seconds |
| `smartctl -a /dev/sdX` | Disk health (SMART data) |
| `echo offline > /sys/block/sdX/device/state` | Emergency: take disk offline |
| `echo "- - -" > /sys/class/scsi_host/host0/scan` | Rescan SCSI bus |

---

*Tags: storage, troubleshooting, linux, disk, performance*
*Categories: Troubleshooting*
