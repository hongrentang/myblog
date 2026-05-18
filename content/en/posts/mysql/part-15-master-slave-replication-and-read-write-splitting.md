---
title: "MySQL from Beginner to Pro · Part 15: Master-Slave Replication and Read-Write Splitting"
date: 2025-01-19
weight: 1315
draft: false
tags: ["mysql"]
---

## What is Master-Slave Replication?

Master-slave replication is the process of synchronizing data changes from the master database to the slave database.

```
Master → binlog → Slave → relay log → data changes
```

**Benefits of Master-Slave Replication:**

| Benefit | Description |
|------|------|
| Read-write splitting | Master for writes, slaves for reads, distributing load |
| Data backup | Slave can serve as real-time backup |
| High availability | Can switch to slave if master fails |
| Data analysis | BI/report queries on slaves, not affecting the master |

---

## 1. Replication Principle

```
Master
┌──────────────────────┐
│ 1. Transaction commit│  binlog
│    → write to binlog │───────┐
└──────────────────────┘       │
                               │
                        Slave IO Thread
                               │
                          ┌────┴─────────┐
                          │ 2. Fetch     │
                          │    binlog    │
                          │    → relay   │
                          │    log       │
                          └────┬─────────┘
                               │
                        Slave SQL Thread
                               │
                          ┌────┴─────────┐
                          │ 3. Replay    │
                          │    relay log │
                          │    locally   │
                          └──────────────┘
```

### Three Threads

| Thread | Location | Role |
|------|------|------|
| Binlog dump thread | Master | Sends binlog to slave |
| IO thread | Slave | Receives binlog and writes to relay log |
| SQL thread | Slave | Reads relay log and replays it |

---

## 2. Configuring Master-Slave Replication

### Master Configuration

```ini
# my.cnf (Master)
[mysqld]
server-id = 1                    # Unique ID (must differ between master and slave)
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW              # ROW format is safest
binlog_row_image = FULL
expire_logs_days = 7
# To replicate specific databases
# binlog-do-db = shop
# binlog-ignore-db = mysql
```

```bash
# Restart master
sudo systemctl restart mysql
```

### Creating Replication User

```sql
-- Log into master
CREATE USER 'repl'@'192.168.1.%' IDENTIFIED BY 'repl_password_123!';
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.1.%';
FLUSH PRIVILEGES;

-- View master status (record File and Position)
SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+------------------+----------+--------------+------------------+
| mysql-bin.000003 |     1234 |              |                  |
+------------------+----------+--------------+------------------+
```

### Slave Configuration

```ini
# my.cnf (Slave)
[mysqld]
server-id = 2                    # Must differ from master
log_bin = /var/log/mysql/mysql-bin.log  # Slave can also enable binlog
relay_log = /var/log/mysql/mysql-relay-bin.log
read_only = 1                    # Slave is read-only (prevents accidental writes)
relay_log_recovery = 1           # Relay log crash recovery
```

```bash
sudo systemctl restart mysql
```

### Starting Replication

```sql
-- Log into slave
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.100',
    MASTER_PORT = 3306,
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'repl_password_123!',
    MASTER_LOG_FILE = 'mysql-bin.000003',
    MASTER_LOG_POS = 1234,
    MASTER_CONNECT_RETRY = 10,
    MASTER_RETRY_COUNT = 0;

-- Start replication
START SLAVE;

-- Check replication status
SHOW SLAVE STATUS\G
```

### Verifying Replication Status

```sql
SHOW SLAVE STATUS\G
```

Key fields:

```
Slave_IO_State: Waiting for master to send event     -- IO thread status
Slave_IO_Running: Yes                                 -- IO thread running
Slave_SQL_Running: Yes                                -- SQL thread running
Seconds_Behind_Master: 0                              -- Replication lag in seconds (0 is best)
Last_IO_Errno: 0                                      -- IO thread error code
Last_SQL_Errno: 0                                     -- SQL thread error code
```

**Normal status**:
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`
- `Seconds_Behind_Master: 0` (ideal; brief non-zero is acceptable)

---

## 3. GTID Replication (MySQL 5.6+)

GTID (Global Transaction Identifier) is a globally unique ID for each transaction, simplifying master-slave switching and failure recovery.

### Enabling GTID

```ini
# my.cnf (configure on both master and slave)
[mysqld]
gtid_mode = ON
enforce_gtid_consistency = ON
```

### GTID-Based Replication Configuration

```sql
-- Configure on slave
CHANGE MASTER TO
    MASTER_HOST = '192.168.1.100',
    MASTER_PORT = 3306,
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'repl_password_123!',
    MASTER_AUTO_POSITION = 1;  -- Use GTID auto-positioning

START SLAVE;
```

GTID replication advantages:
- No need to manually specify `MASTER_LOG_FILE` and `MASTER_LOG_POS`
- On master-switch, the new master's GTID set is automatically passed to the new slave
- After replication interruption, automatically skips already-executed transactions via GTID

---

## 4. Semi-Synchronous Replication

In default asynchronous replication, the master commits a transaction and immediately returns to the client without waiting for slave acknowledgment. If the master crashes, committed transactions may not have been synchronized to the slave.

**Semi-synchronous replication**: The master waits for at least one slave to confirm receipt of the binlog before committing.

```sql
-- Install plugins (master and slave)
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- Enable on master
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 5000;  -- 5 second timeout (falls back to async)

-- Enable on slave
SET GLOBAL rpl_semi_sync_slave_enabled = 1;

-- Restart replication
STOP SLAVE IO_THREAD;
START SLAVE IO_THREAD;
```

### Permanent Configuration

```ini
# my.cnf (Master)
[mysqld]
rpl_semi_sync_master_enabled = 1
rpl_semi_sync_master_timeout = 5000

# my.cnf (Slave)
rpl_semi_sync_slave_enabled = 1
```

---

## 5. Read-Write Splitting

After configuring master-slave replication, read-write splitting needs to be implemented at the application layer.

### Application Layer Implementation (Recommended)

```python
# Python example (using SQLAlchemy)

from sqlalchemy import create_engine
from sqlalchemy.orm import Session

# Master (write)
master_engine = create_engine('mysql://user:pass@192.168.1.100:3306/shop')
# Slave (read)
slave_engine = create_engine('mysql://user:pass@192.168.1.101:3306/shop')

def get_db(read_only=False):
    if read_only:
        return Session(bind=slave_engine)
    return Session(bind=master_engine)

# Usage
db = get_db(read_only=True)    # Reads go to slave
users = db.query(User).all()

db = get_db(read_only=False)   # Writes go to master
db.add(new_user)
db.commit()
```

### ProxySQL Middleware

```bash
# Install ProxySQL
sudo apt install proxysql
sudo systemctl start proxysql
```

```sql
-- Configure ProxySQL (via admin interface on port 6032)
mysql -u admin -padmin -h 127.0.0.1 -P 6032

-- Add backend servers
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES
    (0, '192.168.1.100', 3306),  -- Master (write)
    (1, '192.168.1.101', 3306),  -- Slave 1 (read)
    (1, '192.168.1.102', 3306);  -- Slave 2 (read)

-- Configure read-write splitting rules
INSERT INTO mysql_query_rules
    (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES
    (1, 1, '^SELECT.*', 1, 1),          -- SELECT goes to slave
    (2, 1, '^INSERT|UPDATE|DELETE.*', 0, 1),  -- Writes go to master
    (3, 1, '^SELECT.*FOR UPDATE', 0, 1);      -- FOR UPDATE goes to master

-- Load configuration
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

---

## 6. Common Issues and Solutions

### Replication Interruption

**Error Type 1: SQL Thread Error**

```sql
-- View error
SHOW SLAVE STATUS\G
-- Last_SQL_Error: Could not execute ...  # Usually master-slave data inconsistency

-- Skip a specified number of errors (emergency)
STOP SLAVE;
SET GLOBAL sql_slave_skip_counter = 1;
START SLAVE;

-- Or skip specific error types
-- Set in my.cnf:
-- slave_skip_errors = 1062  # Skip duplicate key errors
-- slave_skip_errors = 1032  # Skip update/delete on non-existent records
```

Better approach: Find and fix the inconsistent data instead of just skipping.

**Error Type 2: IO Thread Error**

```
Last_IO_Error: error reconnecting to master
```

Check:
1. Is the master reachable?
2. Is the replication user password correct?
3. Has the master's binlog been purged?

### Master-Slave Lag

```sql
-- Check lag
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 120  -- 120 seconds behind
```

Common causes of lag:

| Cause | Solution |
|------|---------|
| Slave has lower specs | Upgrade slave hardware |
| Large transactions on master | Split into smaller transactions |
| Slow queries on slave | Optimize slave queries |
| Network latency | Deploy master and slave in the same data center |

### Master-Slave Switch

```sql
-- Promote slave to master
STOP SLAVE;
RESET SLAVE ALL;
-- The slave is now an independent master

-- With GTID, switching is especially simple
-- The original slave can accept writes directly
```

---

## 7. One-Master-Multi-Slave and Master-Master Replication

### One-Master-Multi-Slave Architecture

```
              Master
            /  |  \
          Slave Slave Slave
```

You can have one master with three, five, or more slaves. More slaves means higher read capacity, but also more IO load on the master copying to all slaves.

### Master-Master Replication (Bidirectional)

Two databases act as master and slave to each other; writes to either side are synchronized to the other.

**Not recommended for general use** -- prone to data conflicts (e.g., both sides INSERTing the same primary key simultaneously). Suitable for special scenarios like multi-region active-active.

---

## 8. Recommended Replication Architecture

### Single Master Multiple Slaves + Read-Write Splitting (Recommended)

```
App → ProxySQL
       ├─ Master (write)
       └─ Slave 1 (read)
       └─ Slave 2 (read)
       └─ Slave 3 (backup/reporting)
```

### Cascading Replication

```
Master → Slave 1 → Slave 2 → Slave 3
```

Reduces replication load on the master; Slave 1 acts both as a slave and as the master for other slaves. Suitable for scenarios with many slaves.

---

## Summary

Master-slave replication is the foundation of MySQL high availability and read-write splitting. This article covered replication principles, GTID, semi-sync, read-write splitting solutions, and common issue handling. A well-designed replication architecture enables the database to support higher concurrency and more reliable operation.

---

