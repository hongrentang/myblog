---
title: "Disk Full But du Shows Nothing — When Deleted Files Refuse to Release Their Space"
date: 2026-06-11
weight: 100520
slug: "disk-full-du-shows-no-data"
tags: ["linux", "storage", "troubleshooting", "disk", "filesystem"]
categories: ["Troubleshooting"]
description: "A classic Linux storage mystery — the disk is 100% full according to df, but du shows only 30% used. A Java application had opened a 70GB log file, rotated it (deleted), but never closed the file descriptor, keeping the space allocated"
keywords: "disk full du shows different, deleted file still using space, lsof deleted files, linux disk space mystery, df vs du discrepancy"
draft: false
featured: true
cover:
  image: ""
  caption: "Disk Full But du Shows Nothing — Troubleshooting"
---

# Disk Full But du Shows Nothing — When Deleted Files Refuse to Release Their Space

## Common Search Queries

- disk full but du shows no large files
- df and du show different disk usage
- deleted file still using disk space
- lsof +L1 shows deleted files
- No space left on device but du shows free space
- Linux file descriptor leak disk space
- logrotate deleted file still holding space
- copytruncate vs create logrotate

## The Incident

### Environment

- **OS**: CentOS 7.9, kernel 3.10.x
- **Application**: Java 11 microservice (Spring Boot)
- **Web Server**: Nginx 1.16
- **Logging**: syslog-ng, logrotate daily rotation
- **Monitoring**: Prometheus + Node Exporter + Alertmanager
- **Partition**: `/var` mounted on a separate 200GB logical volume

### Timeline

14:23 — Prometheus Alertmanager fired: `DiskUsage > 90%` for `/var` on the production node. Severity: critical.

14:25 — Operations team logged in to check.

14:26 — `df -h /var` confirmed: `/var` was at **100%** usage. 200GB partition, 0 available blocks.

14:28 — `du -sh /var/* | sort -rh | head -10` showed only about **60GB** of identified data. Where was the other 140GB?

14:30 — Applications began failing. Nginx returned **502 Bad Gateway** because it could not write access logs. Java services threw `java.io.IOException: No space left on device`. The log collector stopped processing.

14:32 — Service degradation declared. Incident response initiated.

### Symptoms

| Symptom | Detail |
|---|---|
| `df -h /var` | 100% full (200G / 200G) |
| `du -sh /var` | ~60G (only 30% used) |
| Inode usage (`df -i`) | Normal, < 5% used |
| App errors | `No space left on device` |
| Nginx | 502 Bad Gateway |
| Prometheus | `node_filesystem_avail_bytes{...} 0` |

## Background

To understand this mystery, we need a quick refresher on how Linux manages files.

### Inodes, Directory Entries, and File Descriptors

Every file on Linux has three layers of metadata:

1. **Directory Entry (dentry)** — The human-readable name in a directory listing. This is what `ls` shows you. It maps a filename to an inode number.

2. **Inode** — The metadata structure that stores all information about a file except its name: size, permissions, timestamps, and **block pointers** (which data blocks on disk belong to this file). Each inode has a **link count** — the number of directory entries pointing to it.

3. **Data Blocks** — The actual file content stored on disk.

When a process opens a file, the kernel creates a **file descriptor (FD)** in the process's `/proc/<pid>/fd/` namespace. This FD holds a reference to the inode, independent of any directory entry.

### How df and du Differ

- **`du`** (disk usage) walks the **directory tree**. It follows directory entries to find inodes, then sums up the data blocks referenced by each inode. If a file has no directory entry (it is unlinked), `du` cannot see it.

- **`df`** (disk free) reads the **filesystem superblock**, which tracks total blocks, used blocks, and free blocks at the block allocator level. It counts all allocated blocks regardless of whether they have a directory entry.

This is the fundamental reason they disagree: `du` sees what is linked in the directory tree; `df` sees what is actually allocated on disk.

### What Happens When a File Is Deleted While Open

When you `rm` a file:

1. The kernel removes the directory entry (unlinks it from the parent directory).
2. The inode's link count is decremented by 1.
3. If the link count reaches **0** and no process holds an open FD to the inode, the inode and its data blocks are freed.
4. But if a process **still has the file open**, the link count goes to 0 yet the inode remains allocated because the FD holds a reference. The data blocks stay on disk, invisible to `du` but fully counted by `df`.

This is exactly the scenario we were dealing with. The file was "deleted" from the directory's perspective, but its data blocks were still alive and well, held hostage by an open file descriptor.

## Investigation

### Step 1: Confirm Disk Full

```bash
df -h /var
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/var 200G  200G     0 100% /var
```

200GB full, 0 available. The alert was real.

### Step 2: Walk the Directory Tree with du

```bash
du -sh /var/* | sort -rh | head -10
```

```
30G     /var/log
12G     /var/lib
8G      /var/cache
5G      /var/opt
3G      /var/spool
1G      /var/tmp
0.5G    /var/www
...
```

Total was around 60GB. A 140GB gap — something was consuming space but not visible in any directory.

### Step 3: Check Inode Usage

Inode exhaustion was a known possibility (see previous incident), but we quickly ruled it out:

```bash
df -i /var
```

```
Filesystem       Inodes  IUsed   IFree IUse% Mounted on
/dev/mapper/var   13M    600K    12.4M    5% /var
```

Only 5% inode usage. The problem was definitely block allocation, not inode exhaustion.

### Step 4: Find Deleted Files Still Held Open

The classic tool for this is `lsof` with the `+L1` flag, which lists files with a link count of 0 (unlinked but still open):

```bash
lsof +L1 /var
```

```
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF   NLINK   NODE NAME
java     17234   root   12w   REG  253,0 72543272960   0  1835009 /var/log/java/app.log.1 (deleted)
```

There it was. A 70GB+ file, link count 0 (deleted), but still open by the Java process.

Alternatively, you can use `/proc` to find the same:

```bash
find /proc/*/fd -ilname "/var/*" 2>/dev/null | while read fd; do
  if [ ! -e "$fd" ]; then
    ls -la "$fd"
  fi
done
```

```
lrwx------ 1 root root 64 Jun 11 14:28 /proc/17234/fd/12 -> /var/log/java/app.log.1 (deleted)
```

### Step 5: Inspect the Java Process File Descriptors

```bash
lsof -p 17234 | grep deleted | head -10
```

```
java  17234  root  12w  REG  253,0  72543272960  0  1835009  /var/log/java/app.log.1 (deleted)
```

The file descriptor number was **12**, the file size was **72,543,272,960 bytes** (roughly 67.5 GiB / 72.5 GB).

We also checked the raw symlink:

```bash
ls -la /proc/17234/fd/ | grep deleted
```

```
l-wx------ 1 root root 64 Jun 11 14:28 12 -> /var/log/java/app.log.1 (deleted)
```

### Step 6: Calculate Total Space Held by Deleted FDs

```bash
lsof -p 17234 | grep deleted | awk '{print $7}' | paste -sd+ | bc
```

```
72543272960
lsof +L1 /var | sort -rn -k7 | head -5
```

This confirmed about **72.5 GB** was being held by deleted-but-open files. This exactly explained the discrepancy: 60GB (visible by du) + 72.5GB (held by deleted FD) = 132.5GB, and the remaining was buffer/cache overhead and other small allocations.

### Step 7: Check Logrotate Configuration

```bash
cat /etc/logrotate.d/java-app
```

```
/var/log/java/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0644 root root
    postrotate
        /bin/kill -HUP `cat /var/run/java-app.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

The logrotate config was using the default **create** strategy: rename the old log, create a new empty file. The postrotate script attempted to send SIGHUP to the Java process to trigger log file reopening — but the Java application did **not** handle SIGHUP, so the old file descriptor remained open.

### Step 8: Check Kernel File Handle Usage

```bash
cat /proc/sys/fs/file-nr
```

```
9984   0   197583
```

Allocated: 9,984 | Unused: 0 | Max: 197,583. Nothing alarming here, though the investigation was thorough.

## Root Cause

The root cause chain is straightforward:

1. The Java microservice opened `/var/log/java/app.log` for writing at startup.
2. Every day at midnight, logrotate renamed `app.log` to `app.log.1` and created a fresh `app.log`.
3. logrotate then ran the postrotate script, which sent SIGHUP to the Java process.
4. The Java process **did not implement a SIGHUP handler** — the signal was ignored, and the process continued writing to the old file handle, which now pointed to `app.log.1`.
5. When logrotate compressed `app.log.1` on the next rotation (delaycompress keeps one uncompressed), the old handle still pointed to the uncompressed data.
6. Over time, `app.log.1` grew to 72.5 GB. When logrotate next ran, it rotated `app.log.1` to `app.log.2.gz`, but the Java FD still pointed to the *unlinked inode* of the original `app.log.1`, which was now fully detached from any directory entry.
7. The file became invisible to `du` but the kernel kept all 72.5 GB of data blocks allocated because the FD was still open.

### Why copytruncate Would Have Prevented This

The key difference:

| Strategy | Mechanism | Risk |
|---|---|---|
| **create** (default) | Rename old log, create new empty file | Requires application to reopen log file |
| **copytruncate** | Copy file content to rotated log, then truncate original to 0 | Application FD still points to the same inode — after truncate, space is released immediately |

With `copytruncate`, the file is never unlinked. The inode stays alive, the FD stays valid, and `truncate` simply frees the data blocks. No application cooperation needed.

## Resolution

### Emergency Fix — Recover Disk Immediately

#### Option A: Restart the Java Service (Recommended)

```bash
systemctl restart java-app
```

After restart:

```bash
df -h /var
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/mapper/var 200G  60G   140G  30% /var
```

The disk dropped to 30% immediately. The 72.5 GB of data blocks were freed when the last file descriptor to the unlinked inode was closed.

#### Option B: Kill -HUP (If App Handles It)

```bash
kill -HUP 17234
```

In our case this did nothing because the Java app did not handle SIGHUP. Check your application documentation.

#### Option C: Truncate the File Descriptor Directly (Zero-Downtime)

If restarting the process is not an option (e.g., can't afford downtime), you can truncate the file directly through `/proc`:

```bash
truncate -s 0 /proc/17234/fd/12
```

```bash
df -h /var
```

This frees the space immediately without restarting the process. However, the application may behave unpredictably if it discovers the file was truncated from under it (the file position stays at the original offset, so it will write a huge hole and then start filling at offset 0 again). Use with caution and test in your environment first.

#### Verify Recovery

```bash
df -h /var
lsof +L1 /var | wc -l   # should be 0
```

### Long-Term Prevention

#### 1. Use copytruncate in logrotate

```bash
cat > /etc/logrotate.d/java-app << 'EOF'
/var/log/java/*.log {
    copytruncate
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
}
EOF
```

This is the most reliable fix. No application changes needed. `copytruncate` copies the log file content to the archive, then truncates the original in-place. The application's file descriptor stays valid, and the space is released immediately upon truncation.

**Caveat**: `copytruncate` has a very small window where log data written between the copy and the truncate could be lost. For most applications this is acceptable; for zero-loss requirements, see option 2.

#### 2. Configure Java Logging to Reopen on SIGHUP

For Logback (Spring Boot default), add a `ConfigurationEventListener` that reopens files on SIGHUP, or configure the `FixedWindowRollingPolicy` to work with external rotation. The Java side should cooperate with logrotate.

#### 3. Add FD Monitoring

Monitor for lingering deleted file descriptors:

```bash
# Nagios / Icinga check
lsof +L1 /var | awk '{sum+=$7} END {print sum}'  # total bytes held by deleted FDs

# Alert if any deleted FD exists
lsof +L1 /var > /dev/null 2>&1 && echo "WARNING: Deleted FDs found"
```

Prometheus rule idea (if using node_exporter, this requires a custom textfile collector):

```yaml
# /etc/prometheus/rules/disk.yml
groups:
  - name: disk
    rules:
      - alert: DeletedFileDescriptorHoldingSpace
        expr: (node_filesystem_size_bytes{...} - node_filesystem_free_bytes{...}) - node_filesystem_avail_bytes{...} > 10 * 1024 * 1024 * 1024
        for: 10m
        annotations:
          summary: "Discrepancy between df and du suggests deleted files holding space"
```

#### 4. Monitor df vs du Discrepancy

Run a simple check on a schedule:

```bash
#!/bin/bash
# /usr/local/bin/check-df-du-discrepancy.sh
MOUNT=$1
THRESHOLD_PCT=${2:-10}
DF_USED=$(df "$MOUNT" | tail -1 | awk '{print $3}')
DU_USED=$(du -s "$MOUNT" | awk '{print $1}')
DIFF=$(( (DF_USED - DU_USED) * 100 / DF_USED ))
if [ "$DIFF" -gt "$THRESHOLD_PCT" ]; then
  echo "CRITICAL: df ($DF_USED) vs du ($DU_USED) differs by ${DIFF}% on $MOUNT"
  exit 2
fi
echo "OK: df vs du within ${THRESHOLD_PCT}% on $MOUNT"
exit 0
```

#### 5. Update Runbook

Document the step-by-step procedure for `df vs du` discrepancy so any on-call engineer can follow it without prior knowledge of this edge case.

## Lessons Learned

1. **df and du measure different things.** Never assume they should match. `df` reports block-level allocation; `du` reports directory-tree usage. A large discrepancy always means allocated blocks without directory entries.

2. **logrotate's default behavior is dangerous for long-running processes.** The `create` strategy (rename + new file) silently breaks if the application does not reopen its log files after rotation. Always verify that your application handles SIGHUP, or use `copytruncate`.

3. **Signal handling matters.** The Java application was sending logs to a file opened at JVM startup. The JVM's default SIGHUP behavior is to exit (on Oracle JDK) or ignore (on some OpenJDK builds). Neither is correct for a production service that needs log rotation. Test SIGHUP handling as part of your deployment checklist.

4. **Monitoring the right metrics.** `df -h` alone is insufficient. Monitor:
   - `df -h` (partition usage)
   - `df -i` (inode usage)
   - `lsof +L1` count (deleted-but-open files, the canary in the coal mine)
   - `df vs du` discrepancy (early warning for this class of issue)

5. **incident response runbooks save time.** An hour of investigation could have been 10 minutes if the on-call engineer had a documented procedure for "disk full but du disagrees."

6. **Kernel keeps its promises.** When Linux says "the data is still there," it means it — even when the file is "deleted." The kernel holds allocated blocks until the last file descriptor is closed. This is correct behavior, not a bug, but it can be surprising.

## Summary

### Timeline

| Time | Event |
|---|---|
| 00:00 | logrotate runs, rotates `app.log` to `app.log.1`, creates new `app.log` |
| 00:00 | Java process ignores SIGHUP, continues writing to old FD (now `app.log.1`) |
| Days pass | `app.log.1` grows to 72.5 GB; logrotate compresses on subsequent nights but the FD on the original inode persists |
| 14:23 | Prometheus alerts: `/var` 100% full |
| 14:25 | Operations team logs in |
| 14:26 | `df -h` confirms 100% |
| 14:28 | `du -sh /var/*` shows only 60GB — the mystery is spotted |
| 14:32 | `lsof +L1 /var` reveals the 72.5 GB deleted FD |
| 14:35 | `systemctl restart java-app` — disk drops to 30% |
| 14:36 | Service restored. Incident resolved. |

### Command Comparison Table

| Command | What It Shows | When to Use |
|---|---|---|
| `df -h` | Block-level allocation (total blocks, used blocks) | First check for disk full |
| `df -i` | Inode allocation count | When df -h shows free space but apps say "no space" |
| `du -sh <path>` | Directory-tree aggregated usage | To find which directories are using space |
| `lsof +L1 <mount>` | Files with link count = 0 (deleted but open) | When df and du disagree significantly |
| `find /proc/*/fd -ilname` | Same as lsof +L1, via /proc | When lsof is not installed |
| `ls -la /proc/<pid>/fd/` | List all open FDs for a process | To inspect a specific process |
| `truncate -s 0 /proc/<pid>/fd/<n>` | (Action) truncate FD to 0 | Emergency space recovery without restart |
| `cat /proc/sys/fs/file-nr` | Kernel FD allocation stats | When you suspect FD exhaustion |

### Key Takeaway

When `df` shows the disk is full but `du` disagrees, you are almost certainly looking at a process that holds an open file descriptor to a deleted file. The fix is to identify the process with `lsof +L1`, then restart it or truncate the FD. For long-term prevention, configure logrotate with `copytruncate` or ensure your application properly handles log rotation signals.
