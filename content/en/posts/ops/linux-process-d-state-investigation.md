---
title: "Linux Process D State — When Processes Freeze and Nothing Can Wake Them Up"
date: 2026-06-12
weight: 100540
slug: "linux-process-d-state-investigation"
tags: ["linux", "kernel", "troubleshooting", "process", "system"]
categories: ["Troubleshooting"]
description: "A Linux process D state (uninterruptible sleep) incident — how a storage controller driver bug caused dozens of processes to hang in D state, freezing the production server and requiring an emergency reboot to recover"
keywords: "linux d state uninterruptible sleep, process stuck in D state, linux hung process, blocked task, kernel hung task timeout"
draft: false
featured: true
cover:
  image: ""
  caption: "Linux Process D State — Troubleshooting"
---

# Linux Process D State — When Processes Freeze and Nothing Can Wake Them Up

## Common Search Queries

If you arrived here from a search engine, you were likely searching for one of these:

- linux process stuck in D state uninterruptible sleep
- how to fix D state process linux
- process D state kill -9 not working
- linux load average high D state processes
- hung_task_timeout_secs blocked for more than 120 seconds
- nvme driver hang D state recovery
- kernel hung task watchdog trigger
- how to check what a D state process is waiting on
- D state vs Z state linux process
- emergency reboot D state processes linux

---

## The Incident

**Environment:**

| Component | Version |
|-----------|---------|
| OS | Ubuntu 22.04 LTS |
| Kernel | 5.15.0-86-generic |
| Storage | NVMe SSD (Intel P5510) |
| Filesystem | ext4 |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| App Runtime | Node.js 18 |

**Timeline:**

It was a quiet Tuesday afternoon. The pg_cron job on PostgreSQL fired its hourly VACUUM cycle — nothing unusual. Then, at 14:37, monitoring alerts started firing.

**Symptoms:**

- SSH connection succeeds, but every command takes 30+ seconds to complete
- `ls /` takes 30 seconds to return
- `ps aux` shows 30+ processes in `D` state
- Ping to the server responds, but TCP connections (HTTP, PostgreSQL client connections) time out
- Load average exceeds 100 on a 32-core machine
- Kernel hung task messages appear in `dmesg`

The server was technically "alive" but practically dead. No new PostgreSQL connections could be established. The Node.js app was returning 502 errors from upstream. Redis was still responding to local commands but couldn't persist to disk.

---

## Background

### Linux Process States

Every Linux process exists in one of several states:

| State | Name | Description |
|-------|------|-------------|
| R | Running / Runnable | Process is actively running or waiting for CPU time |
| S | Interruptible Sleep | Process is waiting for an event (e.g., network data) and can be interrupted by signals |
| D | Uninterruptible Sleep | Process is waiting for I/O completion and cannot be interrupted |
| T | Stopped | Process has been stopped (SIGSTOP/SIGTSTP) |
| Z | Zombie | Process has terminated but its exit code has not been read by its parent |

### What D State Means

D state stands for "Uninterruptible Sleep." When a process is in D state, it is blocked waiting for a specific I/O operation to complete. The kernel has put this process to sleep, and **no signal can wake it** — not SIGTERM, not SIGKILL (signal 9). The process will only leave D state when the I/O operation finishes (successfully or with an error).

This mechanism exists because the kernel must guarantee that certain I/O operations complete atomically. If a process were allowed to be killed while holding a kernel lock or while a storage write is in flight, the kernel could end up in an inconsistent state — corrupted filesystem metadata, lost data, or a kernel panic.

### D State vs Z State

These two are frequently confused:

- **D state (Uninterruptible Sleep):** The process is still alive. It is blocked on I/O. It is NOT a zombie. The process has not exited — it cannot exit because the kernel won't let it until the I/O finishes.
- **Z state (Zombie):** The process has already exited (terminated). Only its process descriptor remains in the process table waiting for the parent to call `waitpid()`.

A D state process consumes CPU resources (it appears in the load average calculation). A Z state process does not consume CPU but does consume a PID table entry.

### Kernel Hung Task Detection

The Linux kernel has a built-in watchdog mechanism for detecting tasks stuck in D state. The `hung_task` kernel thread runs periodically (every `hung_task_check_interval_secs`, default 120 seconds) and checks whether any task has been in D state for longer than `kernel.hung_task_timeout_secs` (default 120 seconds).

When it detects a stuck task, it logs a message like:

```
INFO: task postgres:1234 blocked for more than 120 seconds.
      Tainted: P           O      5.15.0-86-generic
"echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
```

By default, the hung task detector only **logs** the warning — it does not take any recovery action. However, some distributions configure the kernel to trigger a panic or a crash dump (`hung_task_panic = 1`).

---

## Investigation

When the server is partially hung but SSH still works, you have a narrow window to collect diagnostic data. Every command you run competes for the same I/O resources that are already blocked. Be surgical.

### Step 1: Assess the Damage

First, confirm the scope of the problem:

```bash
uptime
```

```
 14:38:12 up 187 days,  3:42,  1 user,  load average: 124.37, 89.52, 42.18
```

Load average of 124 on a 32-core machine. Something is very wrong.

```bash
top
```

In `top`, look at the process state summary line. If it shows dozens of processes in `D` state, you have a systemic I/O hang.

### Step 2: Identify D State Processes

List all processes currently in D state:

```bash
ps -eo pid,stat,wchan,comm | grep "^ *[0-9] D"
```

Sample output:

```
 2341 D     ?          postgres: checkpointer
 2392 D     ?          postgres: walwriter
 2456 D     ?          postgres: autovacuum worker
 2678 D     ?          redis-server
 3123 D     ?          node app-server
 3190 D     ?          node worker
 ...
```

The `wchan` column shows the kernel function the process is blocked on. If they all show the same or similar functions (e.g., `nvme_wait_ready`, `blk_mq_get_tag`, `io_schedule`), the hang is systemic and likely at the block device or driver level.

Alternatively:

```bash
ps aux | awk '$8 ~ /D/ {print}'
```

### Step 3: Check Kernel Hung Task Messages

```bash
dmesg | grep -i "hung_task\|blocked for more than\|D state" | tail -20
```

Look for patterns like:

```
[1034567.890] INFO: task postgres:2456 blocked for more than 120 seconds.
[1034567.890] "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
[1034567.890] postgres       D    0  2456   2345 0x00000000
[1034567.890] Call Trace:
[1034567.890]  __schedule+0x2d6/0x960
[1034567.890]  schedule+0x4b/0xc0
[1034567.890]  io_schedule+0x42/0x70
[1034567.890]  wait_on_page_bit+0xe0/0x130
[1034567.890]  __filemap_fdatawait_range+0x121/0x190
[1034567.890]  filemap_fdatawait+0x25/0x30
[1034567.891]  ext4_sync_file+0x8d/0x1c0
[1034567.891]  do_fsync+0x38/0x60
[1034567.891]  __x64_sys_fsync+0x10/0x20
[1034567.891]  do_syscall_64+0x43/0x90
[1034567.891]  entry_SYSCALL_64_after_hwframe+0x62/0xcc
[1034567.891] 
[1034567.892] INFO: task redis-server:2678 blocked for more than 120 seconds.
[1034567.892] ... (similar call trace)
```

The key observation here is that **all** blocked tasks are stuck in `io_schedule` or `wait_on_page_bit`. This tells you the hang is at the I/O layer, not in application code. They are all waiting for the block layer to complete a read or write operation.

### Step 4: Check What Each D Process Is Waiting On

For a targeted view of exactly what kernel function a specific process is blocked on:

```bash
cat /proc/<pid>/stack
```

Example:

```
[<0>] io_schedule+0x42/0x70
[<0>] wait_on_page_bit+0xe0/0x130
[<0>] __filemap_fdatawait_range+0x121/0x190
[<0>] filemap_fdatawait+0x25/0x30
[<0>] ext4_sync_file+0x8d/0x1c0
[<0>] do_fsync+0x38/0x60
```

This confirms the process is waiting for a page to be written to disk (ext4 filesystem layer), which is waiting for the block layer to complete the write.

Also:

```bash
cat /proc/<pid>/wchan
```

This gives you the raw kernel function name:

```
io_schedule
```

If ALL D state processes show `io_schedule`, `blk_mq_get_tag`, or `nvme_wait_ready`, the problem is at the NVMe driver or block layer level.

### Step 5: Check I/O Stats

```bash
iostat -x 1
```

When the system is hung, `iostat` may also hang or show extremely high `await` and `svctm` values. If `iostat` itself hangs, skip the live tool and read the raw block device stats:

```bash
cat /sys/block/nvme0n1/stat
```

The stat file contains 11 fields. Focus on:

- Field 3 (rd_ticks): total milliseconds spent reading
- Field 7 (wr_ticks): total milliseconds spent writing
- Field 10 (io_ticks): total milliseconds the device has had I/O requests queued

If `io_ticks` is much larger than the uptime, the device has been continuously busy — or stuck.

### Step 6: Check NVMe Driver Status

```bash
cat /sys/class/nvme/nvme0/device/state
```

Expected output:

```
live
```

If it shows something else (e.g., `dead`, `error`), the driver has detected an error state and entered recovery.

```bash
cat /sys/class/nvme/nvme0/device/controller/cntlid
```

Check the NVMe error log:

```bash
nvme list
```

```bash
nvme error-log /dev/nvme0n1 | head -20
```

The error log may show:

```
Error Log Entries for device:nvme0 entries:64
 .................
 Entry[ 0]   
 .................
 error_count     : 3
 sqid            : 0
 cmdid           : 0x003e
 status_field    : 0x4004 (INVALID SQ ID)
 phase_tag       : 0
 parm_err_loc    : 0x0000
 lba             : 0x00000000000000ff
 nsid            : 0x00000001
 vs              : 0
```

The `INVALID SQ ID` error is a red flag — it indicates the driver tried to submit a command to a submission queue that does not exist or has been deactivated.

### Step 7: Check Kernel Logs for NVMe-Specific Errors

```bash
dmesg | grep -i "nvme\|pci\|msi\|irq" | tail -20
```

Look for messages like:

```
nvme nvme0: I/O 214 QID 2 timeout, aborting
nvme nvme0: I/O 215 QID 2 timeout, aborting
nvme nvme0: I/O 216 QID 2 timeout, aborting
nvme nvme0: Abort status: 0x1000
nvme nvme0: Unable to abort command 214
nvme nvme0: controller is down; will reset: LST=0 MST=0
nvme nvme0: Device not ready; aborting reset, CSTS=0x0
...
nvme nvme0: Removing after probe failure status: -19
```

The pattern "Unable to abort command" followed by "controller is down" and "Device not ready" is diagnostic of a driver-level hang where the NVMe controller stopped responding to interrupts entirely.

### Step 8: Check Interrupt Configuration

```bash
cat /proc/interrupts | grep nvme
```

```
  CPU0   CPU1   CPU2   ...  CPU31
 12345      0      0    ...      0  nvme0q0, nvme0q1
     0  23456      0    ...      0  nvme0q2
     0      0  18293    ...      0  nvme0q3
  ...
```

If one or more queues show zero interrupts while others are counting normally, the interrupt for that queue has been lost. This is the smoking gun.

```bash
cat /sys/class/nvme/nvme0/device/irq
```

This shows the IRQ numbers assigned to the NVMe device. Cross-reference with `/proc/interrupts` to see which vectors are active.

---

## Root Cause

After collecting all of the above data, the root cause became clear. Here is the chain of events:

### 1. PostgreSQL VACUUM Triggers Heavy I/O

PostgreSQL's autovacuum worker process initiated a VACUUM on a large table (approximately 120 GB). The VACUUM process performs sequential scans and issues writebacks for dirty pages. PostgreSQL uses `fsync()` to ensure data integrity, which forces the kernel to synchronously write pages to disk.

On this particular workload, the VACUUM triggered over 256 I/O requests in flight simultaneously, saturating the NVMe queue depth.

### 2. MSI-X Interrupt Vector Exhaustion

The Intel P5510 NVMe SSD supports up to 128 MSI-X interrupt vectors. The NVMe driver allocates one interrupt vector per I/O queue. When the workload generated I/O requests across all available queues simultaneously, the MSI-X vector table on the NVMe controller reached its limit.

The NVMe specification allows for queue depth up to 64K commands per queue, but the interrupt vectors are a finite hardware resource. When all vectors are in use and a new interrupt needs to be generated, the controller must reuse an existing vector — and if the timing is wrong, the completion interrupt can be lost.

### 3. Lost Completion Interrupt

A race condition in the `nvme` kernel driver (present in kernel 5.15.0-86) caused a specific scenario:

1. The NVMe driver submits a read command via I/O queue QID 2
2. The command completes on the device, and the device raises an MSI-X interrupt on vector 17
3. Due to a race condition in the interrupt handler (the vector was being reassigned to a different queue), the interrupt is not delivered to any CPU
4. The command is marked as "in flight" in the driver's internal tracking table
5. The driver never receives the completion, so it never frees the command slot

### 4. Block Layer Backpressure

The Linux block layer implements a tag-based system to limit the number of in-flight I/O requests per queue. Each request is assigned a "tag" (an identifier). When all tags are consumed by commands that will never complete, the block layer cannot issue any new I/O requests to that queue.

With the I/O queue completely blocked:

- Every subsequent `read()`, `write()`, and `fsync()` system call enters `io_schedule()` waiting for a tag to become available
- No tag will ever become available because the lost command will never complete
- All processes waiting for I/O enter D state and stay there permanently

### 5. Hung Task Watchdog Fires but Cannot Recover

After 120 seconds, the hung task watchdog logs the "blocked for more than 120 seconds" message. But the watchdog cannot fix the problem because:

- The kernel lock protecting the I/O queue is held by the stuck NVMe command
- The hung task detector only logs warnings — it does not terminate processes (that would be unsafe for D state)
- Even if it tried to kill a D state process, the kernel would refuse because the process is in an uninterruptible sleep

### 6. No Escalation Without Driver Reset

The only way out of this state is:

- The NVMe driver detects the timeout and resets the controller (this can happen automatically)
- An administrator triggers a manual driver reset via sysfs
- A full system reboot

In this incident, the driver's automatic timeout detection also failed because the MSI-X vector exhaustion caused the timeout check itself to hang — a cascading failure.

---

## Resolution

### Emergency Response

The server was in a production environment with active users. Recovery needed to happen immediately.

#### Attempt 1: NVMe Driver Reset

```bash
echo 1 > /sys/class/nvme/nvme0/device/reset_controller
```

Or using the `nvme-cli` tool:

```bash
nvme reset /dev/nvme0n1
```

If the reset command succeeds, you will see kernel messages like:

```
nvme nvme0: resetting controller
nvme nvme0: 32/0/0 default/read/poll queues
nvme nvme0: new controller alive
```

After the reset, the block layer recovers, pending I/Os complete (or fail), and D state processes return to normal.

In this case, the reset command also hung because the kernel was too busy to process the sysfs write. The driver's internal state machine was in an unrecoverable condition.

#### Attempt 2: Emergency Reboot

When the driver reset fails, only a reboot can recover the system.

The safest way to reboot a hung system is using the SysRq key:

```bash
echo b > /proc/sysrq-trigger
```

If `/proc/sysrq-trigger` does not exist or returns "Operation not permitted," enable SysRq first:

```bash
echo 1 > /proc/sys/kernel/sysrq
echo b > /proc/sysrq-trigger
```

The `echo b` SysRq command performs the following:

1. Syncs all mounted filesystems (if possible)
2. Remounts filesystems read-only (if possible)
3. Triggers a hard reset of the CPU

This is significantly safer than pressing the physical reset button because the kernel gets a chance to sync filesystems first.

After the reboot, verify:

```bash
nvme list
```

```
Node                  SN                   Model                                    Namespace Usage                      Format           FW Rev
----------------     --------------------  ----------------------------------------  --------- -------------------------- ---------------- --------
/dev/nvme0n1         PHL0123456789         INTEL SSDPE2KX040T8                      1           4.00  TB /   4.00  TB      512   B +  0B   QDV1B5
```

```bash
dmesg | grep -i nvme | tail -10
```

```
nvme nvme0: 32/0/0 default/read/poll queues
nvme nvme0: new controller alive
nvme nvme0: NVME_CMD_SCSI 0x00, I/O 0 QID 0
```

And confirm no D state processes remain:

```bash
ps -eo stat,pid,comm | grep D
```

The output should be empty.

### Long-Term Prevention

After recovering the server, implement the following measures to prevent recurrence:

#### 1. Update Kernel and NVMe Driver

The bug was fixed in kernel 5.15.0-92 (and fully resolved in 5.15.0-100+). Upgrade to a kernel version that includes the NVMe MSI-X race condition fix.

For Ubuntu 22.04:

```bash
sudo apt update
sudo apt install --install-recommends linux-generic-hwe-22.04
sudo reboot
```

#### 2. Reduce NVMe Timeouts

The default NVMe timeout is 60 seconds for admin commands and 30 seconds for I/O commands. Reduce these to detect hangs faster:

Add to kernel boot parameters (in `/etc/default/grub`):

```
GRUB_CMDLINE_LINUX_DEFAULT="... nvme_core.admin_timeout=30 nvme_core.io_timeout=30"
```

Then:

```bash
sudo update-grub
sudo reboot
```

#### 3. Create NVMe Module Configuration

Create `/etc/modprobe.d/nvme.conf`:

```
options nvme_core io_timeout=30 admin_timeout=30 max_host_queues=8
```

The `max_host_queues=8` parameter limits the number of I/O queues, reducing the probability of hitting the MSI-X vector limit (at the cost of some parallelism).

#### 4. Increase Hung Task Threshold

The default 120-second hung task timeout may be too aggressive for systems with high I/O latency. Increase it:

```bash
sysctl -w kernel.hung_task_timeout_secs=300
```

Make it permanent:

```bash
echo "kernel.hung_task_timeout_secs=300" >> /etc/sysctl.d/99-hung-task.conf
```

#### 5. Set Up Monitoring

Monitor D state process count. A quick script:

```bash
#!/bin/bash
# /usr/local/bin/check_d_state.sh
D_COUNT=$(ps -eo stat | grep -c "^D")
if [ "$D_COUNT" -gt 5 ]; then
    echo "CRITICAL: $D_COUNT processes in D state | d_state=$D_COUNT"
    exit 2
elif [ "$D_COUNT" -gt 2 ]; then
    echo "WARNING: $D_COUNT processes in D state | d_state=$D_COUNT"
    exit 1
fi
echo "OK: $D_COUNT processes in D state | d_state=$D_COUNT"
```

For Prometheus-based monitoring, add a node-level metric:

```
node_process_state{state="D"}
```

Configure alerting: alert if 5+ processes are in D state for 5+ minutes.

#### 6. Configure Kernel Crash Dump

Set up kdump to capture kernel memory if the hung task watchdog detects a fatal hang:

```bash
sudo apt install linux-crashdump
sudo dpkg-reconfigure linux-crashdump
```

Then configure `hung_task_panic`:

```bash
sysctl -w kernel.hung_task_panic=1
```

This causes the kernel to panic when a hung task is detected, triggering kdump to capture a crash dump for post-mortem analysis.

#### 7. Document Recovery Procedure

Document the escalation path:

| Step | Action | Command |
|------|--------|---------|
| 1 | Check D state processes | `ps -eo pid,stat,wchan,comm \| grep "^ *[0-9] D"` |
| 2 | Check kernel stack | `cat /proc/<pid>/stack` |
| 3 | Check NVMe driver state | `cat /sys/class/nvme/nvme0/device/state` |
| 4 | Check NVMe error log | `nvme error-log /dev/nvme0n1` |
| 5 | Reset NVMe controller | `echo 1 > /sys/class/nvme/nvme0/device/reset_controller` |
| 6 | Emergency reboot | `echo b > /proc/sysrq-trigger` |
| 7 | Verify recovery | `ps -eo stat,pid,comm \| grep D` |

---

## Lessons Learned

### What Went Well

- SSH remained functional despite the system being nearly hung. The kernel scheduler still gave CPU time to the SSH daemon, allowing diagnostic access.
- Kernel hung task messages provided clear call traces pointing directly to the I/O layer.
- The NVMe error log persisted across the reboot, allowing post-mortem analysis.

### What Could Have Gone Better

- **No kdump was configured.** When the driver entered the infinite loop, there was no crash dump to analyze the exact code path. If kdump had been configured, the root cause analysis could have been completed in hours instead of days.
- **Monitoring gap.** D state process count was not monitored. If it had been, the alert would have fired 120 seconds into the incident (when the hung task watchdog first fired), not 15 minutes later when TCP connections started timing out.
- **No timeout parameter tuning.** The NVMe driver was running with stock timeout values. Reducing the I/O timeout from 30 seconds to 15 seconds would not have prevented the hang, but it would have made the driver detect the lost interrupt faster and initiate recovery sooner.
- **Single point of failure.** The PostgreSQL data directory and the Redis append-only log were on the same NVMe device. A single driver bug took out both the database and the cache simultaneously. Separating these onto different storage devices (or at minimum different NVMe namespaces) would have limited the blast radius.

### Key Takeaways for Operations Teams

1. **D state is not always recoverable without a reboot.** While many D state hangs resolve on their own (e.g., a slow disk eventually completes the I/O), a driver-level bug can make D state permanent. Be prepared to execute an emergency reboot.

2. **SysRq is your friend.** The Magic SysRq key (`echo b > /proc/sysrq-trigger`) is dramatically safer than a hard power cycle. It gives the kernel a chance to sync filesystems.

3. **Monitor D state process count.** A single D state process is usually normal (database checkpointer, kernel worker). Five or more is a red flag. Ten or more is a crisis.

4. **Tune kernel hung task parameters.** Default 120 seconds may be too long for latency-sensitive systems. Reduce or increase based on your workload.

5. **Storage driver bugs are rare but catastrophic.** The NVMe driver is generally rock-solid, but like all kernel code, it has bugs. When they trigger, the entire I/O stack can freeze. Redundant storage paths and regular kernel updates are your best defense.

---

## Summary

### Incident Timeline

| Time (UTC+8) | Event |
|-------------|-------|
| 14:37:00 | PostgreSQL autovacuum starts on a 120 GB table |
| 14:37:12 | NVMe driver MSI-X race condition triggers; first completion interrupt lost |
| 14:37:15 | Block layer tags exhausted; all new I/O blocked |
| 14:37:20 | 30+ processes enter D state; load average spikes |
| 14:38:00 | First monitoring alerts (Node.js app health check fails) |
| 14:39:12 | Hung task watchdog fires; dmesg logs blocked tasks |
| 14:42:00 | SSH diagnostics begin |
| 14:45:00 | NVMe driver reset attempted — hangs |
| 14:46:15 | Emergency SysRq reboot initiated |
| 14:48:30 | Server back online; PostgreSQL recovery completes in WAL replay |
| 15:00:00 | All services restored |

### Command Reference

| Diagnostic Goal | Command |
|----------------|---------|
| List D state processes | `ps -eo pid,stat,wchan,comm \| grep "^ *[0-9] D"` |
| Count D state processes | `ps -eo stat \| grep -c "^D"` |
| View kernel stack of a process | `cat /proc/<pid>/stack` |
| View kernel function blocked on | `cat /proc/<pid>/wchan` |
| View all D state process stacks | `for p in $(ps -eo pid,stat \| awk '/D/ {print $1}'); do echo "=== PID $p ==="; cat /proc/$p/stack 2>/dev/null; done` |
| Check kernel hung task messages | `dmesg \| grep -i "hung_task\|blocked for more than"` |
| Check NVMe device state | `cat /sys/class/nvme/nvme0/device/state` |
| Check NVMe interrupt assignment | `cat /proc/interrupts \| grep nvme` |
| Check NVMe error log | `nvme error-log /dev/nvme0n1` |
| Reset NVMe driver | `echo 1 > /sys/class/nvme/nvme0/device/reset_controller` or `nvme reset /dev/nvme0n1` |
| Emergency reboot (safe) | `echo b > /proc/sysrq-trigger` |
| Configure hung task timeout | `sysctl -w kernel.hung_task_timeout_secs=300` |
| List block device stats | `cat /sys/block/nvme0n1/stat` |

### Final Thoughts

The Linux D state is one of the most misunderstood process states in system administration. It is not a "stuck" state in the normal sense — it is the kernel's way of saying "I have committed to this I/O operation and I cannot go back." When it works correctly, it protects data integrity. When a driver bug causes it to become permanent, it can bring an entire server to its knees.

The most important takeaway: **you cannot kill a D state process.** Do not waste time trying `kill -9`. Find the I/O device it is waiting on, fix the device or driver, and only then will the processes recover.
