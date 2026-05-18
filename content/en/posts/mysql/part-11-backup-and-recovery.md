---
title: "MySQL from Beginner to Pro · Part 11: Backup and Recovery"
date: 2025-01-15
weight: 1311
draft: false
tags: ["mysql"]
---

## The Importance of Backup

```
Before backups: Two kinds of people — those who haven't had an accident and those who have
After backups: Two kinds of people — those who can recover and those who can't
```

A production database without backups is only one wrong operation away from disaster.

**What You Should Do:**

- Regular backups (full + binlog)
- Regular backup verification (can the backup file be successfully restored)
- Offsite storage (don't keep backups on the same machine as the database)

---

## 1. Backup Types

| Type | Description | Speed | Restore Speed | Storage |
|------|------|------|---------|------|
| **Logical Backup** | SQL statement format (mysqldump) | Slow | Slow | Compressible, small size |
| **Physical Backup** | Direct file copy | Fast | Fast | Not compressible, large size |

| Method | Description | Use Case |
|------|------|---------|
| **Full Backup** | Backup all data | Periodic (e.g., daily at midnight) |
| **Incremental Backup** | Backup data changed since last backup | Systems with frequent changes |
| **binlog Backup** | Backup binary logs | Point-in-time recovery |

---

## 2. mysqldump Logical Backup

### Basic Usage

```bash
# Backup a single database
mysqldump -u root -p shop > shop_backup.sql

# Backup multiple databases
mysqldump -u root -p --databases shop blog > multi_backup.sql

# Backup all databases
mysqldump -u root -p --all-databases > all_backup.sql

# Backup a single table
mysqldump -u root -p shop users > users_backup.sql
```

### Common Parameters

```bash
# Include stored procedures and events (not included by default)
mysqldump -u root -p --routines --events shop > backup.sql

# Backup structure only
mysqldump -u root -p --no-data shop > schema.sql

# Backup data only (without CREATE TABLE statements)
mysqldump -u root -p --no-create-info shop > data.sql

# Compressed backup (recommended)
mysqldump -u root -p shop | gzip > shop_$(date +%Y%m%d).sql.gz

# Specify character set
mysqldump -u root -p --default-character-set=utf8mb4 shop > backup.sql
```

### Recommended Production Parameters

```bash
mysqldump -u root -p \
  --single-transaction \       # Lock-free backup (uses InnoDB transaction)
  --routines \                 # Include stored procedures
  --events \                   # Include events
  --triggers \                 # Include triggers
  --master-data=2 \            # Record binlog position (for master-slave setup)
  --set-gtid-purged=ON \       # GTID information
  --skip-lock-tables \         # Don't lock tables (with single-transaction)
  --quick \                    # Read row by row, avoid excessive memory usage
  shop | gzip > shop_$(date +%Y%m%d_%H%M%S).sql.gz
```

> **`--single-transaction`** is the key parameter for InnoDB -- it creates a consistent snapshot at the start of the transaction. DML operations from other transactions during the backup won't affect the backup content, and no table locking is needed (MyISAM still requires table locking; confirm all your tables are InnoDB before using this parameter).

---

## 3. Backup Restore

```bash
# Restore a full backup
mysql -u root -p shop < shop_backup.sql

# Restore from compressed file
gunzip < shop_20250115.sql.gz | mysql -u root -p shop

# Restore multiple databases
mysql -u root -p < multi_backup.sql

# Restore all databases
mysql -u root -p < all_backup.sql
```

### Partial Restore

```bash
# Restore a single table from a full backup
sed -n '/^-- Table structure for table `users`/,/^-- Table structure for table/p' backup.sql > users.sql
mysql -u root -p shop < users.sql

# More precise method: use parameters to filter tables during mysqldump
mysqldump -u root -p shop users orders > partial.sql
```

---

## 4. Physical Backup

### XtraBackup (Recommended)

Percona XtraBackup supports **hot backup** for InnoDB without blocking reads and writes.

```bash
# Install
sudo apt install percona-xtrabackup-84

# Full backup
xtrabackup --backup \
  --user=root \
  --password=xxx \
  --target-dir=/backups/full/

# Prepare backup (apply redo log for consistency)
xtrabackup --prepare --target-dir=/backups/full/

# Restore
systemctl stop mysql
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backups/full/
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql

# Incremental backup
xtrabackup --backup \
  --target-dir=/backups/inc1 \
  --incremental-basedir=/backups/full/

# Merge incremental into full
xtrabackup --prepare \
  --apply-log-only \
  --target-dir=/backups/full/
xtrabackup --prepare \
  --target-dir=/backups/full/ \
  --incremental-dir=/backups/inc1/
```

### Direct File Copy

```bash
# Stop MySQL and copy the data directory directly (cold backup)
systemctl stop mysql
cp -r /var/lib/mysql /backups/mysql_cold_$(date +%Y%m%d)
systemctl start mysql
```

---

## 5. binlog and Point-in-Time Recovery

binlog (binary log) records all data change operations and is the key to **incremental recovery**.

### Enabling binlog

```ini
# my.cnf
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_format = ROW          # ROW format recommended
expire_logs_days = 7         # Retain 7 days
binlog_row_image = FULL      # Record all columns (easier for recovery)
```

### Viewing binlog

```bash
# View all binlog files
mysql -u root -p -e "SHOW BINARY LOGS;"

# View the current binlog being written
mysql -u root -p -e "SHOW MASTER STATUS;"

# View binlog contents
mysqlbinlog /var/log/mysql/mysql-bin.000023 > binlog.sql

# View by time range
mysqlbinlog \
  --start-datetime="2025-01-15 10:00:00" \
  --stop-datetime="2025-01-15 11:00:00" \
  /var/log/mysql/mysql-bin.000023 > recover.sql
```

### Complete Recovery Process

**Scenario**: A user accidentally DELETEd a table at 10:30.

```bash
# 1. Restore the most recent full backup
mysql -u root -p shop < shop_20250115.sql

# 2. Replay the binlog after the backup (skip the time of the accident)
mysqlbinlog \
  --start-datetime="2025-01-15 03:00:00" \
  --stop-datetime="2025-01-15 10:29:59" \
  /var/log/mysql/mysql-bin.* | mysql -u root -p shop

# Or find the log position of the mistaken operation and recover by position
mysqlbinlog \
  --start-position=12345 \
  --stop-position=67890 \
  /var/log/mysql/mysql-bin.* | mysql -u root -p shop
```

### Position-Based Recovery

```bash
# View binlog to find the mistaken operation position
mysqlbinlog /var/log/mysql/mysql-bin.000023 | grep -n "DROP TABLE\|DELETE FROM users"

# Recover by position
mysqlbinlog \
  --stop-position=12345 \   # Stop before the mistaken operation
  /var/log/mysql/mysql-bin.000023 | mysql -u root -p shop

# If one position was already hit, skip and continue
mysqlbinlog \
  --start-position=67890 \   # Start after the mistaken operation
  /var/log/mysql/mysql-bin.000023 | mysql -u root -p shop
```

---

## 6. Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/mysql_backup.sh

DB_USER="root"
DB_PASS="your_password"
DB_NAME="shop"
BACKUP_DIR="/backups/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup directory
mkdir -p "$BACKUP_DIR"

# mysqldump backup
echo "[$(date)] Starting backup of $DB_NAME..."
mysqldump \
  --user="$DB_USER" \
  --password="$DB_PASS" \
  --single-transaction \
  --routines \
  --events \
  --triggers \
  --master-data=2 \
  "$DB_NAME" | gzip > "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

# Check if backup succeeded
if [ $? -eq 0 ] && [ -s "$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz" ]; then
    echo "[$(date)] Backup completed: ${BACKUP_DIR}/${DB_NAME}_${DATE}.sql.gz"
else
    echo "[$(date)] Backup FAILED!"
    exit 1
fi

# Delete old backups
find "$BACKUP_DIR" -name "${DB_NAME}_*.sql.gz" -mtime +$RETENTION_DAYS -delete

# Sync to remote location (optional)
# rsync -avz "$BACKUP_DIR/" user@remote-server:/backups/mysql/
```

**Using with crontab**:

```bash
# Backup daily at 3:00 AM
0 3 * * * /usr/local/bin/mysql_backup.sh
```

---

## 7. Backup Verification

> A backup that hasn't been tested for restore is not a backup

```bash
#!/bin/bash
# Periodic backup verification

BACKUP_FILE="/backups/mysql/shop_20250115.sql.gz"
TEST_DB="restore_test"

# Create test database
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS $TEST_DB"

# Restore backup to test database
gunzip < "$BACKUP_FILE" | mysql -u root -p $TEST_DB

# Check for errors
ERROR_COUNT=$(mysql -u root -p $TEST_DB -e "SHOW TABLE STATUS" 2>&1 | grep -c "Error")

if [ "$ERROR_COUNT" -eq 0 ]; then
    echo "[$(date)] Backup verification PASSED"
else
    echo "[$(date)] Backup verification FAILED: $ERROR_COUNT errors"
fi

# Cleanup
mysql -u root -p -e "DROP DATABASE $TEST_DB"
```

---

## 8. Best Practices Summary

```
Backup Golden Rules:
┌────────────────────────────────────────────────────┐
│ 1. Regular full backups (at least daily)           │
│ 2. Enable binlog (for point-in-time recovery)      │
│ 3. Store backups separately from the database      │
│ 4. Regularly verify backup integrity               │
│ 5. Keep a copy of your backup offsite              │
│ 6. Practice restore procedures (don't wait for a real disaster) │
└────────────────────────────────────────────────────┘
```

| Backup Strategy | Use Case | RPO (Data Loss) | RTO (Recovery Time) |
|---------|---------|----------------|--------------|
| Daily full backup | Small data | At most 1 day of data loss | Hours |
| Daily full + binlog | Production systems | At most a few seconds | Hours |
| XtraBackup full + incremental | Large data | Depends on incremental frequency | Tens of minutes |

---

## Summary

A database without backups is incomplete. This article covered mysqldump, XtraBackup, binlog recovery, and automated backup scripts, covering the complete workflow from daily backups to disaster recovery.

---

