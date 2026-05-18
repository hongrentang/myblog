---
title: "MySQL from Beginner to Pro · Part 12: User Privileges and Security"
date: 2025-01-16
weight: 1312
draft: false
tags: ["mysql"]
---

## 1. User Management

### Viewing Users

```sql
SELECT User, Host, plugin, authentication_string
FROM mysql.user;

-- Formatted view
SELECT
    User,
    Host,
    account_locked,
    password_expired,
    password_last_changed
FROM mysql.user;
```

### Creating Users

```sql
-- Basic creation
CREATE USER 'app_user'@'localhost' IDENTIFIED BY 'secure_password_123';

-- Allow connection from any IP
CREATE USER 'app_user'@'%' IDENTIFIED BY 'secure_password_123';

-- Restrict to subnet
CREATE USER 'admin'@'192.168.1.%' IDENTIFIED BY 'another_password';

-- Change password after creation
ALTER USER 'app_user'@'localhost' IDENTIFIED BY 'new_password';
```

### Deleting Users

```sql
DROP USER 'old_user'@'localhost';
DROP USER IF EXISTS 'old_user'@'localhost';
```

---

## 2. Privilege System

### Privilege Hierarchy

```
Global privileges (*.*)
  └── Database privileges (db.*)
        └── Table privileges (db.table)
              └── Column privileges (db.table.col)
```

### Granting Privileges

```sql
-- Grant all privileges on a database (excluding GRANT privilege)
GRANT ALL PRIVILEGES ON shop.* TO 'app_user'@'localhost';

-- Read-only privileges
GRANT SELECT ON shop.* TO 'readonly_user'@'%';

-- Precise privileges (common combination)
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app_user'@'%';

-- Table creation and modification privileges (DBA only)
GRANT CREATE, ALTER, DROP, INDEX ON shop.* TO 'dba_user'@'localhost';

-- Stored procedure privileges
GRANT EXECUTE ON PROCEDURE shop.get_user_orders TO 'app_user'@'%';

-- Column-level grants (special scenarios)
GRANT SELECT (id, name, email) ON shop.users TO 'analyst'@'%';

-- Always execute this last
FLUSH PRIVILEGES;
```

### Revoking Privileges

```sql
-- Revoke all write privileges
REVOKE INSERT, UPDATE, DELETE ON shop.* FROM 'app_user'@'%';

-- Revoke all privileges
REVOKE ALL PRIVILEGES ON shop.* FROM 'user'@'%';

-- Flush after revocation
FLUSH PRIVILEGES;
```

### Viewing Privileges

```sql
-- View current user privileges
SHOW GRANTS;

-- View privileges of a specific user
SHOW GRANTS FOR 'app_user'@'localhost';

-- View global privileges of current user
SELECT * FROM mysql.user WHERE user = 'root'\G
```

### Common Privilege Combinations

| Role Type | Privileges | Description |
|---------|------|------|
| Read-only account | `SELECT` | For BI tools, data analysis |
| Application account | `SELECT, INSERT, UPDATE, DELETE` | Application connection |
| DBA account | `ALL PRIVILEGES` | Database administrator, add `WITH GRANT OPTION` |
| Backup account | `SELECT, LOCK TABLES, RELOAD, REPLICATION CLIENT` | Backup-specific |
| Replication account | `REPLICATION SLAVE, REPLICATION CLIENT` | Slave connection |

```sql
-- Application account example
CREATE USER 'app'@'192.168.10.%' IDENTIFIED BY 'app_password123!';
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app'@'192.168.10.%';
FLUSH PRIVILEGES;

-- Read-only account example
CREATE USER 'readonly'@'%' IDENTIFIED BY 'readonly_pass!';
GRANT SELECT ON shop.* TO 'readonly'@'%';
FLUSH PRIVILEGES;
```

---

## 3. Password Policy

### View Current Policy

```sql
SHOW VARIABLES LIKE 'validate_password%';
```

### Setting Password Policy (requires validate_password component)

```sql
-- Enable password validation component
INSTALL COMPONENT 'file://component_validate_password';

-- Set password policy level (0=low, 1=medium, 2=strong)
SET GLOBAL validate_password.policy = MEDIUM;

-- Minimum password length
SET GLOBAL validate_password.length = 12;

-- Require uppercase, lowercase, numbers, special characters
SET GLOBAL validate_password.mixed_case_count = 1;
SET GLOBAL validate_password.number_count = 1;
SET GLOBAL validate_password.special_char_count = 1;
```

### Password Expiration Policy

```sql
-- Set user password to expire in 90 days
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE INTERVAL 90 DAY;

-- Password never expires (not recommended)
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE NEVER;

-- Force user to change password on next login
ALTER USER 'app_user'@'localhost' PASSWORD EXPIRE;

-- Set global password expiration policy (in my.cnf)
-- default_password_lifetime = 90
```

---

## 4. Connection Security

### Limiting Connections

```sql
-- Limit user's maximum connections
ALTER USER 'app_user'@'%' WITH MAX_USER_CONNECTIONS 50;

-- Global max connections
-- Set in my.cnf:
-- max_connections = 500
```

### Bind Address

```ini
# my.cnf
[mysqld]
# Listen on local connections only
bind-address = 127.0.0.1

# Listen on specific internal IP
bind-address = 192.168.1.100

# Listen on all interfaces (not recommended)
# bind-address = 0.0.0.0
```

---

## 5. SSL/TLS Encrypted Connections

### Configuring MySQL SSL

```ini
# my.cnf
[mysqld]
ssl-ca = /etc/mysql/ssl/ca.pem
ssl-cert = /etc/mysql/ssl/server-cert.pem
ssl-key = /etc/mysql/ssl/server-key.pem

require_secure_transport = ON  # Force SSL usage
```

### Creating Users That Require SSL

```sql
CREATE USER 'secure_user'@'%' IDENTIFIED BY 'password'
REQUIRE SSL;

-- More strict: require X509 certificate
CREATE USER 'cert_user'@'%' IDENTIFIED BY 'password'
REQUIRE X509;
```

### Verifying SSL Connection

```sql
-- Check after connecting
SHOW STATUS LIKE 'Ssl_cipher';
-- A value indicates the current connection is using SSL encryption
```

---

## 6. Audit and Logging

### Auditing Connections

```sql
-- View all current connections
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- View connection sources
SELECT
    user,
    host,
    db,
    command,
    time,
    state,
    info
FROM information_schema.PROCESSLIST;
```

### Query Log

```sql
-- Enable query log (be careful in production! Can write a lot of logs)
SET GLOBAL general_log = ON;
SET GLOBAL general_log_file = '/var/log/mysql/query.log';

-- View recent queries only
SELECT * FROM mysql.general_log
WHERE event_time > NOW() - INTERVAL 5 MINUTE;
```

**Production recommendation**: Use Performance Schema instead of general log for less performance impact:

```sql
SELECT *
FROM performance_schema.events_statements_history
WHERE thread_id = (SELECT thread_id FROM performance_schema.threads
                   WHERE processlist_id = CONNECTION_ID());
```

### Audit Plugin

MySQL Enterprise provides an audit plugin, or use third-party audit plugins.

```ini
# my.cnf (requires audit plugin installation)
[mysqld]
plugin-load-add = audit_log.so
audit_log_policy = LOGINS  # Only log login events
# audit_log_policy = ALL   # Log all operations
```

---

## 7. SQL Injection Defense

> Part 13 of this series will cover SQL injection in detail. This section only covers database-level security configurations.

### Disabling Dangerous Features

```sql
-- Disable LOAD DATA LOCAL INFILE (prevents reading server files)
-- Set in my.cnf:
-- local-infile = 0

-- Disable FILE privilege (don't grant FILE to application accounts)
-- FILE privilege allows using SELECT ... INTO OUTFILE to write files
```

### Minimum Privilege Checklist

```
Database User Privilege Checklist:
□ Application accounts only have SELECT, INSERT, UPDATE, DELETE
□ No DDL privileges (CREATE, ALTER, DROP)
□ No FILE privilege
□ No SUPER privilege
□ No GRANT OPTION
□ Restrict connection source IP
□ Limit max connections
□ Rotate passwords regularly
```

---

## 8. Security Best Practices Summary

### my.cnf Security Configuration

```ini
[mysqld]
# Port and binding
port = 3306
bind-address = 192.168.1.100

# Disable local-infile
local-infile = 0

# Skip symbolic links (prevents linking to sensitive files)
skip_symbolic_links = yes

# Logging
log_error = /var/log/mysql/error.log
slow_query_log = ON
long_query_time = 2

# Password policy
default_password_lifetime = 90
password_history = 10           # Cannot reuse the last 10 passwords
password_reuse_interval = 365   # Cannot reuse passwords within 365 days

# Connection security
max_connect_errors = 100        # Block IP after 100+ connection errors
skip_name_resolve               # Don't resolve hostnames (reduces DNS risk)

# If enabling SSL
# require_secure_transport = ON
```

### Regular Security Tasks

```bash
#!/bin/bash
# Regular security audit script

echo "=== MySQL Security Check $(date) ==="

# 1. Check for passwordless users
mysql -u root -p -e "
SELECT User, Host FROM mysql.user
WHERE authentication_string = '' OR authentication_string IS NULL;
"

# 2. Check for expired passwords
mysql -u root -p -e "
SELECT User, Host, password_expired FROM mysql.user
WHERE password_expired = 'Y';
"

# 3. Check users not using SSL
mysql -u root -p -e "
SELECT User, Host, ssl_type FROM mysql.user
WHERE ssl_type = '';
"

# 4. Check for locked users
mysql -u root -p -e "
SELECT User, Host, account_locked FROM mysql.user
WHERE account_locked = 'Y';
"

echo "=== Check Complete ==="
```

---

## Summary

Security is the bottom line of database management. This article covered user management, privilege systems, password policies, SSL connections, auditing, and best practices. The **principle of least privilege** is the first rule of security -- give users only the minimum permissions needed to do their work.

---

