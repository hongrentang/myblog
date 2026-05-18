---
title: "MySQL from Beginner to Pro · Part 1: MySQL Introduction and Installation"
date: 2025-01-21
weight: 1301
draft: false
tags: ["mysql"]
featured: true
cover:
  image: "/images/mysql-banner.svg"
  caption: "MySQL from Beginner to Pro"
---

## What is MySQL?

MySQL is an open-source relational database management system (RDBMS), originally developed by MySQL AB in Sweden. It was later acquired by Sun Microsystems and eventually became part of Oracle. MySQL is one of the most popular open-source databases in the world, especially common in web applications.

**Core Features:**

- **Relational Database**: Data is organized in tables, with relationships between tables
- **SQL Standard**: Supports standard SQL syntax
- **ACID Transactions**: Supports Atomicity, Consistency, Isolation, Durability
- **Master-Slave Replication**: Supports read-write splitting and high availability
- **Pluggable Storage Engines**: InnoDB, MyISAM, Memory, etc.
- **Cross-Platform**: Supports Linux, Windows, macOS

---

## MySQL vs MariaDB

MariaDB is a fork of MySQL created by the original author Monty after MySQL was acquired by Oracle. They are highly compatible:

| Comparison | MySQL | MariaDB |
|------|-------|---------|
| Current Version | 8.4+ / 9.x | 11.x |
| Maintainer | Oracle | MariaDB Foundation |
| Compatibility | — | Fully compatible with MySQL API |
| New Features | HeatWave, JSON enhancements | More storage engines, performance optimizations |

In most scenarios, they are interchangeable. This series uses **MySQL 8.4** as an example.

---

## Installing MySQL

### Ubuntu/Debian

```bash
# Install MySQL 8.4
sudo apt update
sudo apt install mysql-server-8.4 -y

# Check status
sudo systemctl status mysql

# Secure installation (set root password, remove anonymous users, etc.)
sudo mysql_secure_installation
```

### CentOS/RHEL

```bash
# Add official repository
sudo rpm -ivh https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm

# Install
sudo yum install mysql-server -y

# Start
sudo systemctl start mysqld
sudo systemctl enable mysqld

# View temporary root password
sudo grep 'temporary password' /var/log/mysqld.log
```

### Docker Installation

```bash
docker run --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -p 3306:3306 \
  -d mysql:8.4
```

---

## Connecting to MySQL

```bash
# Local connection
mysql -u root -p

# Remote connection
mysql -h 192.168.1.100 -P 3306 -u root -p

# Show version after connecting
mysql> SELECT VERSION();
+-----------+
| VERSION() |
+-----------+
| 8.4.2     |
+-----------+

# View all current databases
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

---

## MySQL Architecture

```
Client Connections (Connection Pool)
    ↓
SQL Interface (Parser → Optimizer → Executor)
    ↓
Storage Engine Layer (InnoDB / MyISAM / Memory ...)
    ↓
File System (Data Files + Log Files)
```

### Key Components

| Component | Role |
|------|------|
| **Connection Manager** | Manages client connections, authenticates identity |
| **Query Parser** | Parses SQL statements, generates parse tree |
| **Query Optimizer** | Selects the optimal execution plan |
| **Storage Engine** | Data storage and retrieval |
| **Buffer Pool** | Caches data and indexes, reduces disk I/O |
| **Log System** | redo log (crash recovery), binlog (replication and recovery) |

---

## Storage Engine Comparison

```sql
-- View supported storage engines
SHOW ENGINES;

-- View current default storage engine
SELECT @@default_storage_engine;
+--------------------------+
| @@default_storage_engine |
+--------------------------+
| InnoDB                   |
+--------------------------+
```

| Feature | InnoDB ✅ | MyISAM ❌ | Memory |
|------|-----------|-----------|--------|
| Transaction Support | ✅ | ❌ | ❌ |
| Row-Level Locking | ✅ | ❌ (Table lock) | ❌ (Table lock) |
| Foreign Keys | ✅ | ❌ | ❌ |
| Crash Recovery | ✅ | ❌ | ❌ |
| MVCC | ✅ | ❌ | ❌ |
| Full-Text Index | ✅ (8.0+) | ✅ | ❌ |

> In MySQL 8.0+, **InnoDB is the default and recommended storage engine**. MyISAM has been marked as deprecated in newer versions.

---

## First Database

```sql
-- Create database
CREATE DATABASE shop;
USE shop;

-- Create table
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Insert data
INSERT INTO users (name, email) VALUES ('张三', 'zhangsan@example.com');

-- Query data
SELECT * FROM users;
+----+--------+------------------------+---------------------+
| id | name   | email                  | created_at          |
+----+--------+------------------------+---------------------+
|  1 | 张三   | zhangsan@example.com   | 2025-01-15 10:00:00 |
+----+--------+------------------------+---------------------+
```

---

## Common Administration Commands

```sql
-- Show MySQL version
SELECT VERSION();

-- Show current user
SELECT USER();

-- Show current database
SELECT DATABASE();

-- Show all databases
SHOW DATABASES;

-- Switch to database
USE database_name;

-- Show all tables in database
SHOW TABLES;

-- Show table structure
DESC table_name;
DESCRIBE table_name;

-- Show CREATE TABLE statement
SHOW CREATE TABLE table_name;

-- Show MySQL configuration
SHOW VARIABLES LIKE '%character%';
SHOW VARIABLES LIKE '%timeout%';
```

---

## Configuration File

MySQL configuration file paths:

```bash
# Linux
/etc/mysql/mysql.conf.d/mysqld.cnf
# or
/etc/my.cnf

# View currently used config file
mysql --help | grep "Default options"
```

Core configuration example:

```ini
[mysqld]
# Basic
port = 3306
bind-address = 0.0.0.0
datadir = /var/lib/mysql

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# Storage engine
default-storage-engine = InnoDB

# Connection
max_connections = 200
wait_timeout = 600

# Buffer pool (recommended 70% of memory)
innodb_buffer_pool_size = 4G

# Logging
log_error = /var/log/mysql/error.log
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# binlog
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 7
```

Restart after modifying configuration:

```bash
sudo systemctl restart mysql
```

---

## Summary

This article covered MySQL installation, connection, architecture, and basic concepts.

---

