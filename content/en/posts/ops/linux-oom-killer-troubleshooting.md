---
title: "Linux OOM Killer — When Your Java Process Vanishes Without a Trace"
date: 2026-05-25
weight: 100180
slug: "linux-oom-killer-troubleshooting"
tags: ["linux", "oom-killer", "java", "memory", "troubleshooting"]
categories: ["Linux"]
description: "Java process vanishes at peak hours — no crash log, no heap dump, no error. From suspecting a code bug to finding OOM Killer in dmesg, then fixing overcommit and protecting critical processes"
keywords: "oom killer, linux out of memory, java process killed, dmesg, overcommit, oom_adj"
draft: false
featured: true
cover:
  image: "/images/oom-killer-banner.svg"
  caption: "Linux OOM Killer Troubleshooting"
---

# Linux OOM Killer — When Your Java Process Vanishes Without a Trace

## The Incident

Wednesday, 2 PM. Alert: "Payment reconciliation process not found."

Not "process crashed" — "process doesn't exist." The monitoring script checked the PID and it was gone.

```bash
ps aux | grep payment-reconciliation
```

Nothing. The process had vanished.

Check the application log:

```
14:02:15.123 [Thread-42] INFO  - Batch 20260525-001: processing record 5000/50000
14:02:15.456 [Thread-42] INFO  - Batch 20260525-001: processing record 5100/50000
# Nothing after this. No errors, no exceptions, no warnings.
```

Logs just stopped at 14:02:15.

```bash
ls -la hs_err_pid*
```

No JVM crash log.

```bash
tail -20 gc.log
```

GC logs look normal. Plenty of heap space.

Environment: 64GB physical RAM, Java heap -Xms32g -Xmx32g, CentOS 7 VM, not containerized.

**Impact**: Reconciliation job failed. Daily billing couldn't be generated. Ops team restarted the process — it worked again. But the next day, same time, it disappeared again. Three days in a row.

## Investigation

### Wrong turn 1: Suspect a code bug

Process dies → check the code. Recent deployments? Nothing. Logs at the end were normal — processing a batch. DB connections, thread pools, queue depth — all fine.

Not a code bug.

### Wrong turn 2: Look for JVM crash log

A Java crash usually leaves `hs_err_pid*.log`:

```bash
find / -name "hs_err_pid*" 2>/dev/null
```

Nothing.

```bash
journalctl -u payment-reconciliation.service | grep -i "killed\|signal\|main process"
```

Service manager thinks the process exited normally (exit code 0).

**Lesson learned**: When the OOM Killer kills a Java process, **the JVM has no idea what happened**. OOM Killer sends SIGKILL directly from the kernel. Signal handlers don't run. No hs_err, no heap dump, no exit log — the process just vanishes. It's the same as typing `kill -9` in a terminal — the JVM is dead before it can write anything.

### Wrong turn 3: Suspect ops activity

Asked everyone — no one ran kill. No deployments. No systemd restarts.

```bash
ausearch -k process_kill -ts 14:00 -te 14:10 2>/dev/null
```

No audit event (auditd wasn't configured for this).

### The real finding: dmesg

When a process vanishes without explanation, the OOM Killer is the only suspect.

```bash
dmesg -T | grep -i "killed process\|oom"
```

```
[Tue May 25 14:02:15 2026] java invoked oom-killer: gfp_mask=0x100cca(GFP_HIGHUSER_MOVABLE), order=0, oom_score_adj=0
[Tue May 25 14:02:15 2026] java: cpuset=/ mems_allowed=0
[Tue May 25 14:02:15 2026] CPU: 15 PID: 2345 Comm: java Killed
[Tue May 25 14:02:15 2026] Call Trace:
[Tue May 25 14:02:15 2026]  dump_header+0x4a/0x196
[Tue May 25 14:02:15 2026]  oom_kill_process+0xe6/0x120
[Tue May 25 14:02:15 2026]  out_of_memory+0x10c/0x260
...
[Tue May 25 14:02:15 2026] Out of memory: Killed process 2345 (java) total-vm:48123456kB, anon-rss:42123456kB, file-rss:560kB, shmem-rss:0kB, UID:1000 pgtables:94520kB oom_score_adj:0
[Tue May 25 14:02:15 2026] oom_reaper: reaped process 2345 (java), now anon-rss:0kB
```

**Confirmed**. OOM Killer killed the Java process at exactly 14:02:15.

Memory state at the time:

```bash
dmesg -T | grep "Memory:"
```

```
Memory: 58570164K/63872844K available (10234K kernel code, ...)
```

62GB total, 57GB used, 5.3GB "available" but still triggered OOM.

Why? Because of Linux's **memory overcommit** mechanism.

## Root Cause

| Layer | Cause |
|-------|-------|
| Direct | Linux OOM Killer determined the system was low on memory and picked the largest consumer (Java) |
| Root cause | `vm.overcommit_memory=0` (default). Kernel allows memory allocation beyond physical limits. When commit limit was nearly reached + a page cache spike happened, OOM was triggered |
| Trigger | Monitoring agent memory leak (~1.2GB) + daily batch job page cache growth (~4GB) + Java was already at 41GB RSS |
| Why same time daily | Monitoring agent ran full scan at 2 PM; memory + page cache peak coincided with daily batch processing |

**Key concept: Linux memory overcommit**

`overcommit_memory` has three modes:

| Value | Meaning | Behavior |
|-------|---------|----------|
| 0 (default) | Heuristic overcommit | Not strictly checked, not fully open. When memory is near exhaustion, OOM Killer fires |
| 1 | Always overcommit | Always allow allocation beyond physical RAM |
| 2 | Don't overcommit | Strict accounting — allocations exceeding `overcommit_ratio` fail |

With `overcommit_memory=0`, when system memory exceeds ~95%, the kernel decides to kill the largest process before things get worse.

**Why Java's virtual memory was 48GB**:

```
Java heap: -Xmx32g → 32GB
Metaspace: ~1GB
Thread stacks (threads × 1MB): ~2GB
Code Cache: ~256MB
Direct Buffer: ~512MB
Native JVM memory: ~1GB
---
Total virtual ~48GB, RSS ~41GB (actual physical)
```

Core lesson: **Java uses far more physical memory than -Xmx**. 32GB heap + JVM overhead easily reaches 40GB+. If total system memory is barely larger than -Xmx (e.g., 64GB with 32GB heap), any burst from other processes triggers OOM.

## Fix

### Emergency: restart + stop monitoring agent

```bash
systemctl start payment-reconciliation
systemctl stop memory-hungry-monitoring-agent
```

Temporary — same thing would happen tomorrow.

### Short-term fix: tune overcommit

```bash
echo 2 > /proc/sys/vm/overcommit_memory
```

Persist:

```
echo "vm.overcommit_memory = 2" >> /etc/sysctl.conf
echo "vm.overcommit_ratio = 90" >> /etc/sysctl.conf
sysctl -p
```

**Why this works**:
- `overcommit_memory=2` makes the kernel reject allocations exceeding `swap + RAM × overcommit_ratio`
- Java tries to allocate 48GB virtual at startup. If the system doesn't have enough commit charge, the JVM fails to start immediately
- "Fail at startup" is infinitely better than "get randomly killed at 2 PM"

### Root fix: reduce Java heap + protect critical processes

```bash
# 1. Reduce heap from 32GB to 24GB
# -Xms24g -Xmx24g → leaves 8GB+ for system overhead
```

```bash
# 2. Fix monitoring agent memory leak
```

```bash
# 3. Protect critical processes with oom_score_adj
# Lower = less likely to be killed. Range: -1000 (protected) to 1000 (preferred victim)
echo -500 > /proc/2345/oom_score_adj
```

In systemd service file:

```
[Service]
ExecStartPre=/bin/bash -c 'echo -500 > /proc/self/oom_score_adj'
ExecStart=/usr/bin/java -Xms24g -Xmx24g -jar app.jar
```

### Verification

```bash
# 1. No more OOM kills
dmesg -T | grep "Killed process" | grep java
# Empty = safe

# 2. Memory within safe range
free -h
```

```
              total        used        free      shared  buff/cache   available
Mem:           62Gi        38Gi        12Gi        1.0Gi        12Gi        12Gi
```

```bash
# 3. Overcommit policy active
cat /proc/sys/vm/overcommit_memory
# 2
```

### Long-term prevention

```bash
# 1. Proactive memory alerting — don't wait for OOM
# Alert when available memory < 10%
free -h | awk 'NR==2{print $7}'
# Alert when < 5GB

# 2. Regular dmesg OOM check
dmesg -T | grep -i "oom-killer\|killed process"

# 3. Monitor monitoring agents
# The irony: the monitoring agent's memory leak is what triggered OOM

# 4. In K8S: always set memory request = limits
# Burstable QoS = first evicted when node pressure hits
```

## What I Learned

1. **Java process "vanished"? Check dmesg first.** No hs_err, no heap dump, no exit code — that pattern is almost always OOM Killer. `dmesg -T | grep -i "killed process"` should be every Java ops engineer's muscle memory.

2. **-Xmx 32G ≠ process uses 32G.** Java's RSS is much larger than the heap. Thread stacks, Metaspace, Direct Buffer, Code Cache, JVM native — add up to 50%+ of heap. In production, never allocate more than 60-70% of physical RAM to Java heap.

3. **overcommit_memory=0 is the most dangerous mode.** It lets processes start successfully with seemingly adequate memory, then assassinates them at runtime. Switch to `overcommit_memory=2` — a hard failure at startup is infinitely better than a mysterious runtime OOM at 2 PM.

4. **OOM Killer doesn't always kill the process that leaked.** Sometimes it kills the biggest process because another process (monitoring agent, sidecar) had a transient spike. `oom_score_adj` can protect critical processes, but the most reliable fix is "ensure the system has enough memory headroom." Dropping Java heap by 8GB to leave breathing room is more effective than any kernel tuning.
