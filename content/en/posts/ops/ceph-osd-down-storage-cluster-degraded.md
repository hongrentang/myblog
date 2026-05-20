---
title: "Ceph OSD Down Causing Storage Cluster Degraded — A Disk Failure Chain Reaction"
date: 2026-05-19
weight: 100050
slug: "ceph-osd-down-storage-cluster-degraded"
tags: ["ceph", "storage", "troubleshooting"]
categories: ["存储"]
description: "Full postmortem of Ceph OSD failures caused by bad disks — from misdiagnosing network issues to finding bad sectors and same-batch disk failures"
keywords: "ceph osd down, ceph pg degraded, ceph storage cluster failure, osd troubleshooting, ceph data recovery"
draft: false
featured: true
cover:
  image: "/images/ceph-osd-down-banner.svg"
  caption: "Ceph OSD Down — Disk Failure Deep Dive"
---

# Ceph OSD Down Causing Storage Cluster Degraded — A Disk Failure Chain Reaction

## Background

Tuesday, around 2 PM. Our Ceph cluster had been running fine for over a year — 3 MONs, 12 OSD nodes (6 HDDs per node), 72 OSDs total. Triple replication pool, latency around 5-10ms normally.

I was in a meeting when alerts started firing. The cluster went from HEALTH_OK to HEALTH_WARN, then HEALTH_ERR within minutes.

```bash
cluster:
    id:     a3f8c2d1-...
    health: HEALTH_ERR
            3 osds down
            47 pgs degraded
            12 pgs stuck stale
            1 pool has unfound objects
```

**Impact**: Triple replication means we can lose up to 2 copies without data loss — but with 3 OSDs down simultaneously, if any PG happened to land all 3 copies on those OSDs, we'd lose data. Worse, Kafka and ES clusters backed by this storage were already reporting IO timeouts.

## Investigation

### Wrong Turn 1: Blamed the Network

Three OSDs down at once — my first thought was a switch failure. We'd seen OSD heartbeat timeouts trigger false positives before.

```bash
ceph osd tree | grep -E "down|osd\."
```

```
-3       10.00  host storage-node-05
 15    1.90   down        1.0000  osd.15
 16    1.90   down        1.0000  osd.16
 42    1.90   down        1.0000  osd.42
```

All three were on the **same node: storage-node-05**. A network switch issue would have hit OSDs across multiple nodes, not just one. That ruled out the network pretty quickly.

### Wrong Turn 2: Thought OSD Processes Hung

SSH'd to storage-node-05:

```bash
ssh storage-node-05
systemctl status ceph-osd@15
```

```
● ceph-osd@15.service - Ceph OSD 15
   Loaded: loaded /usr/lib/systemd/system/ceph-osd@.service
   Active: active (running) since Mon 2026-05-18 22:00:00 CST
  Process: 12345 ExecStart=/usr/bin/ceph-osd -f -i 15 (code=killed, signal=KILL)
 Main PID: 23456 (ceph-osd)
   Status: "HEALTH_OK"
```

Shows active/running — so why does `ceph -s` say it's down? This is where I made my first real mistake. I trusted systemctl status and didn't check the logs for another 10 minutes.

My coworker casually asked "did you check journalctl?" — and that's when things clicked:

```bash
journalctl -u ceph-osd@15 --since "2 hours ago" | tail -50
```

```
May 19 14:15:23 storage-node-05 ceph-osd[12345]: ** ERROR: osd.15 has been marked down by mon.a
May 19 14:15:23 storage-node-05 ceph-osd[12345]: ** or its heartbeat packets are not being received
May 19 14:15:23 storage-node-05 ceph-osd[12345]: ** This could indicate a hardware or kernel issue
May 19 14:15:25 storage-node-05 ceph-osd[12345]: osd.15 192.168.1.105:6800/12345 --> ** MESSAGE DELAY: 380.5s **
May 19 14:15:25 storage-node-05 ceph-osd[12345]: osd.15 192.168.1.105:6800/12345 --> heartbeat timeout from osd.42
```

**Lesson**: systemctl says active doesn't mean the OSD is actually serving data. MON marks OSDs down when heartbeats fail, but the process can still be running. Never trust systemctl status alone.

### Wrong Turn 3: Suspected OOM Killer

I saw "code=killed, signal=KILL" and jumped to the conclusion that the OOM killer had terminated the OSD process. Checked dmesg:

```bash
dmesg | grep -i oom
dmesg | grep -i kill
```

No OOM records. Memory was fine too:

```bash
free -h
```

```
              total        used        free      shared  buff/cache
Mem:          125Gi        48Gi        62Gi        2.3Gi        15Gi
```

Not memory. So what killed it?

### The Real Investigation

Going back through journalctl, I found a detail I'd initially glossed over:

```
May 19 14:10:23 storage-node-05 ceph-osd[12345]: /var/lib/ceph/osd/ceph-15/: ** read error ** at offset 0x7b4a00000, length 4096
May 19 14:10:23 storage-node-05 ceph-osd[12345]: ** ERROR: bluefs_fsync: fsync on /var/lib/ceph/osd/ceph-15//block.wal failed: Input/output error
May 19 14:10:23 storage-node-05 ceph-osd[12345]: ** Fatal: bluefs: during fsync, abort
```

**Input/output error**. This wasn't a software problem — the disk was dying.

```bash
mount | grep ceph-15
```

```
/dev/sdc1 on /var/lib/ceph/osd/ceph-15 type xfs (rw,noatime)
```

SMART check:

```bash
smartctl -a /dev/sdc | grep -E "Reallocated|Pending|Offline|UDMA|Current_Pending"
```

```
  5 Reallocated_Sector_Ct   198   198   140    -    583
197 Current_Pending_Sector   252   252   000    -    1265
198 Offline_Uncorrectable    252   252   000    -    943
```

583 reallocated sectors, 1265 pending. This disk was toast.

But why did three OSDs fail at once? Checked the other two:

```bash
mount | grep -E "ceph-16|ceph-42"
```

sdd and sde. Both also had bad sectors, just not as bad as sdc. Then it hit me — all the HDDs on storage-node-05 were from the same purchase batch, all about 4 years old. They weren't cascading — they were all failing independently, sdc just happened to be worst.

```bash
iostat -x 1 5 | grep -E "sdc|sdd|sde|await"
```

```
Device     r/s     w/s     rkB/s     wkB/s  await  svctm  %util
sdc       0.5     0.2       2.0       1.0  8500    5200   99.8
sdd      12.3     8.1     512.0     340.0  3200    1800   68.5
sde      18.5    12.2     768.0     512.0  1800     950   55.2
```

sdc's await was 8500ms — 8.5 seconds per IO operation. That disk was basically dead.

## Root Cause

1. **Direct cause**: `/dev/sdc` on storage-node-05 had extensive bad sectors, causing OSD.15's WAL writes to fail with IO errors. The OSD process aborted
2. **Parallel failures**: sdd and sde on the same node also had accumulating bad sectors (same batch, ~4 years old). Each disk was independently failing — high IO latency caused their OSD processes to miss heartbeat deadlines, and MON marked them all down
3. **Root cause**: All HDDs on this node were 4+ years old, past the typical 3-5 year replacement window. No regular SMART巡检 (inspection) was in place — bad sectors accumulated silently

**Biggest mistake**: I spent 15 minutes checking the network, checking processes, checking OOM — everything except the disk. If I'd run `journalctl -u ceph-osd@15 | grep "error"` first, I'd have found it in 5 minutes.

## Solution

### Step 1: Drain the Bad OSDs

```bash
ceph osd out osd.15
ceph osd out osd.16
ceph osd out osd.42
```

Wait for PG rebalancing (took ~40 minutes with our data size):

```bash
ceph -w | grep -E "recover|degraded|active+clean"
```

### Step 2: Remove and Replace

```bash
systemctl stop ceph-osd@15
ceph osd crush remove osd.15
ceph auth del osd.15
ceph osd rm osd.15

# Repeat for osd.16, osd.42
```

### Step 3: Add New Disks

```bash
mkfs.xfs /dev/sdc1 -f
ceph-volume lvm create --data /dev/sdc1 --osd-id 15
```

### Verify

```bash
ceph -s
```

```
cluster:
    id:     a3f8c2d1-...
    health: HEALTH_OK
    osd: 72 osds: 72 up, 72 in
    pgs:     1024 active+clean
```

### Long-Term Fixes

1. **Automated SMART巡检** — weekly check, alert if Reallocated_Sector_Ct > 50
2. **Batch disk replacement** — when one disk in a batch fails, replace all same-batch disks
3. **Add NVMe for WAL/DB** — reduce IO pressure on HDDs and improve WAL write reliability. This node didn't have NVMe acceleration before
4. **CRUSH rule audit** — ensure PGs aren't concentrated on same-node OSDs

## What I Learned

1. **journalctl is the first place to check for OSD issues**, not systemctl status. An OSD can be running but marked down
2. **Multiple OSDs down on the same node? Check disks first, not the network.** Same-node failures are almost always hardware
3. **HDDs past 3 years need SMART monitoring**. Reallocated_Sector_Ct > 100 is a red flag. Don't wait for IO errors
4. **Same-batch disks fail around the same time**: when one disk goes, the others from the same batch are likely close behind. Proactive batch replacement prevents surprise multi-OSD failures
5. **Triple replication isn't bulletproof** — if all three replicas happen to land on the same node, that node failing means data loss. Review CRUSH rules regularly
