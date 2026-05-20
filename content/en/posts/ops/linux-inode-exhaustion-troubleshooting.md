---
title: "inode Exhaustion — When the Disk Has Space but Can't Write a Single File"
date: 2026-05-20
weight: 100070
slug: "linux-inode-exhaustion-troubleshooting"
tags: ["linux", "filesystem", "troubleshooting"]
categories: ["Linux"]
description: "Full postmortem of an inode exhaustion failure — from misdiagnosing disk space to finding 1.1M small files in postfix maildrop"
keywords: "inode exhaustion, df -i, no space left on device, linux troubleshooting, postfix maildrop"
draft: false
featured: true
cover:
  image: "/images/linux-inode-exhaustion-banner.svg"
  caption: "Linux inode Exhaustion Troubleshooting"
---

# inode Exhaustion — When the Disk Has Space but Can't Write a Single File

## Background

Tuesday morning. Alert channel exploded — log collection stopped on a batch of servers, all showing "No space left on device."

First reflex — `df -h`:

```bash
df -h
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G   12G   38G  24% /
```

24% used. Where's the "no space"?

But the app was definitely failing. Java processes were throwing `IOException: No space left on device`, the log collector was down. If this dragged on, we'd start losing business data.

## Investigation

### Wrong Turn 1: Suspected Filesystem Corruption

First thought — maybe `df` cache is stale. Ran `sync`, checked again:

```bash
sync
df -h /
```

Same — 24%.

Then I did the dumbest thing — **ran fsck on a production box**.

```bash
# Don't do this in production
fsck -n /dev/vda1
```

At least I used `-n` (check-only). Nothing wrong. But the fsck blocked disk IO for a few seconds, adding latency to production requests for no reason — **a completely unnecessary risky operation**. Wasted 15 minutes.

**Lesson**: "No space" with `df -h` showing free space — don't jump to filesystem corruption. Check **inodes** first.

### Wrong Turn 2: Suspected File Descriptor Exhaustion

"Maybe the process hit the max open file limit?"

```bash
lsof | wc -l
```

```
83521
```

80K+? That seems high. What's the limit?

```bash
ulimit -n
```

```
1024
```

Wait — 80K > 1024? That doesn't add up. Then I remembered: `ulimit -n` is **per-process**, not system-wide. `lsof | wc -l` counts all processes combined. Completely different numbers.

```bash
# Actual per-process fd count
for pid in /proc/[0-9]*; do
    fd_count=$(ls "$pid/fd" 2>/dev/null | wc -l)
    if [ "$fd_count" -gt 1000 ]; then
        echo "$(cat $pid/comm 2>/dev/null) (PID $(basename $pid)): $fd_count fds"
    fi
done | sort -t: -k2 -rn | head -5
```

```
java (PID 12345): 2341 fds
dockerd (PID 987): 1567 fds
```

Java had 2300+ fds — nowhere near the system limit (`cat /proc/sys/fs/file-max` is usually in the millions). File descriptors weren't the problem.

### Wrong Turn 3: Suspected Disk Hardware Failure

"Bad sectors causing IO errors?"

```bash
smartctl -a /dev/vda | grep -E "Reallocated|Pending|Offline"
```

```
  5 Reallocated_Sector_Ct   100   100   010    -    0
197 Current_Pending_Sector   100   100   000    -    0
```

Disk was clean.

```bash
iostat -x 1 3
```

```
Device   r/s   w/s   await  %util
vda      2.3   1.8   1.2    0.8
```

1.2ms latency, <1% util — the disk was practically idle.

This was a purely software-level error. I'd been chasing the wrong direction for 30 minutes.

### The Real Discovery

A coworker glanced at my screen. "Did you check inodes?"

I paused — inodes?

```bash
df -i /
```

```
Filesystem     Inodes  IUsed   IFree IUse% Mounted on
/dev/vda1      3.2M    3.2M      87  100% /
```

**100%**. Inodes exhausted. The filesystem still had 38G of data blocks free, but every inode slot was taken. Any operation that creates a new file returns "No space left on device."

This is the filesystem's two-dimensional space problem: **data blocks AND inodes are both finite. Either one full means "no space."**

Now I needed to find what was eating all the inodes.

```bash
# Find the directory with the most files
for dir in /*; do
    count=$(find "$dir" -xdev 2>/dev/null | wc -l)
    echo "$count $dir"
done | sort -rn | head -5
```

```
1425621 /var
...
```

1.4M files under `/var`. Drill down:

```bash
for dir in /var/*; do
    count=$(find "$dir" -xdev 2>/dev/null | wc -l)
    echo "$count $dir"
done | sort -rn | head -5
```

```
1347289 /var/spool
...
```

1.3M under `/var/spool`. Deeper:

```bash
ls -la /var/spool/postfix/maildrop/ | wc -l
```

```
1123457
```

1.13 million tiny files in `postfix maildrop`. Each one is a small email (a few hundred bytes) generated when a cron job produces output without redirecting it. Each file consumes one inode.

```bash
ls -la /var/spool/postfix/maildrop/ | head -5
```

```
-rw------- 1 postfix postfix 328 May 20 03:47 B1C2D3E4F5
-rw------- 1 postfix postfix 297 May 20 03:48 G6H7I8J9K0
-rw------- 1 postfix postfix 341 May 20 03:49 L1M2N3O4P5
```

300 bytes per file. 1.13 million of them = ~350MB total — negligible for data blocks, but 1.13 million inodes. The entire root partition only had 3.2M inodes total.

## Root Cause

- **Direct cause**: `/var/spool/postfix/maildrop/` accumulated 1.1M+ small files, exhausting all inodes
- **Root cause**: Dozens of cron jobs on the system never redirect their output. By default, cron captures stdout/stderr and attempts to email it to the user via the system MTA (postfix). Every cron run = one tiny email = one inode consumed. Years of this = millions of files
- **Contributing factor**: postfix had no auto-cleanup configured, and there was no inode monitoring

A bad crontab entry looks like this:

```bash
# Bad — cron will capture output and try to email it
*/5 * * * * /usr/local/bin/healthcheck.sh

# Good — redirect to null or a log file
*/5 * * * * /usr/local/bin/healthcheck.sh > /dev/null 2>&1
```

We had dozens of entries like the bad one, running for years.

## Solution

### Emergency: Free Inodes

```bash
# Can't use rm -rf directly — too many files, "Argument list too long"
# Use find + delete instead
find /var/spool/postfix/maildrop/ -type f -delete
```

This took about 8 minutes — 1.1M files to delete, which is itself IO work.

Watch progress in another shell:

```bash
watch -n 5 'df -i /'
```

```
Every 5.0s: df -i /

Filesystem     Inodes  IUsed   IFree IUse% Mounted on
/dev/vda1      3.2M    2.1M   1.1M   66%  /
```

Verify recovery:

```bash
# Confirm inode usage is back to normal
df -i /
df -h /

# Restart the affected service
systemctl restart log-collector
journalctl -u log-collector --since "10 minutes ago" | grep -i error
```

✅ **Recovery verification**:
- `df -i` shows usage below 30%
- Application stops throwing "No space left on device"
- Log collector resumes normal operation

### Long-Term Fix

```bash
# 1. Disable postfix — this server doesn't need to send mail
systemctl stop postfix
systemctl disable postfix

# 2. Or if you need postfix, add periodic cleanup
cat > /etc/cron.daily/clean-maildrop << 'EOF'
#!/bin/bash
find /var/spool/postfix/maildrop/ -type f -mtime +7 -delete
EOF
chmod +x /etc/cron.daily/clean-maildrop

# 3. Find and fix all crontabs without output redirects
grep -r "^[^#].*[a-zA-Z0-9]" /var/spool/cron/crontabs/ | grep -v ">/dev/null" | grep -v ">&/dev/null"
```

### Prevention: Add Inode Monitoring

```bash
# Prometheus alert rule
# node_filesystem_files_free / node_filesystem_files < 0.1
# Alert when inode usage exceeds 90%
```

## What I Learned

1. **"No space left on device" doesn't always mean the disk is full on the `df -h` level**. Filesystems have two finite resources: data blocks and inodes. Always check both: `df -h` AND `df -i`.

2. **Undirected cron output is a silent inode killer**. Without `> /dev/null 2>&1` or a log redirect, every cron run queues an email. Dozens of cron jobs × years of running = millions of tiny files.

3. **Don't run fsck on production as a first diagnostic**. It's slow, risky, and probably won't find anything. Start with the lightest, safest commands — `df -i` takes 0.1 seconds and would have found this immediately.

4. **Small files are an inode tax**. Container logs, temp caches, session files, mail queues — they all consume disproportionate inode counts relative to their disk footprint. Monitoring should track both dimensions.

5. **Disable postfix on servers that don't need it**. Most Linux distributions install and start postfix by default, but most servers never send mail. One `systemctl disable postfix` prevents an entire class of inode exhaustion scenarios.
