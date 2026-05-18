---
title: "MySQL from Beginner to Pro · Part 10: Triggers and Events"
date: 2025-01-14
weight: 10
draft: false
tags: ["mysql"]
---

## 1. Trigger

A trigger is an **automatically executed** stored program -- it fires automatically when INSERT, UPDATE, or DELETE operations are performed on a table.

### Basic Syntax

```sql
CREATE TRIGGER trigger_name
{BEFORE | AFTER} {INSERT | UPDATE | DELETE}
ON table_name
FOR EACH ROW
BEGIN
    -- Trigger logic
END;
```

### A Simple Example

```sql
-- Log delete operations on the users table
DELIMITER //

CREATE TRIGGER before_user_delete
BEFORE DELETE ON users
FOR EACH ROW
BEGIN
    INSERT INTO user_delete_log (user_id, name, email, deleted_at)
    VALUES (OLD.id, OLD.name, OLD.email, NOW());
END //

DELIMITER ;
```

---

### OLD and NEW

| Operation | OLD | NEW |
|------|-----|-----|
| INSERT | ❌ Does not exist | ✅ Newly inserted row |
| UPDATE | ✅ Row before modification | ✅ Row after modification |
| DELETE | ✅ Deleted row | ❌ Does not exist |

```sql
-- Update trigger: log values before and after modification
DELIMITER //

CREATE TRIGGER before_user_update
BEFORE UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO user_change_log
        (user_id, field, old_value, new_value, changed_at)
    VALUES
        (NEW.id, 'email', OLD.email, NEW.email, NOW());

    -- If phone number was also modified, log it too
    IF OLD.phone != NEW.phone THEN
        INSERT INTO user_change_log
            (user_id, field, old_value, new_value, changed_at)
        VALUES
            (NEW.id, 'phone', OLD.phone, NEW.phone, NOW());
    END IF;
END //

DELIMITER ;
```

### Trigger Use Cases

#### 1. Auto-update Timestamp (Simplified)

```sql
CREATE TRIGGER before_users_update
BEFORE UPDATE ON users
FOR EACH ROW
SET NEW.updated_at = NOW();
```

> MySQL 8.0+ can directly use `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` to achieve this without a trigger.

#### 2. Auto-deduct Inventory

```sql
CREATE TRIGGER after_order_insert
AFTER INSERT ON order_items
FOR EACH ROW
BEGIN
    UPDATE products
    SET stock = stock - NEW.quantity
    WHERE id = NEW.product_id;
END;
```

> Note: This scenario is **not recommended** for triggers -- overselling may occur under high concurrency. It's recommended to use `UPDATE ... WHERE stock >= quantity` with locking at the application layer.

#### 3. Data Integrity Check

```sql
CREATE TRIGGER before_order_insert
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    -- Order total cannot be negative
    IF NEW.total_amount < 0 THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = '订单金额不能为负';
    END IF;

    -- Order number is required
    IF NEW.order_no IS NULL OR NEW.order_no = '' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = '订单号不能为空';
    END IF;
END;
```

#### 4. Audit Logging

```sql
CREATE TRIGGER after_user_update
AFTER UPDATE ON users
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (table_name, record_id, action, old_data, new_data, created_at)
    VALUES (
        'users',
        NEW.id,
        'UPDATE',
        JSON_OBJECT('name', OLD.name, 'email', OLD.email),
        JSON_OBJECT('name', NEW.name, 'email', NEW.email),
        NOW()
    );
END;
```

### Trigger Management

```sql
-- View triggers
SHOW TRIGGERS;

-- View trigger definition
SHOW CREATE TRIGGER before_user_delete;

-- Query information_schema
SELECT * FROM information_schema.TRIGGERS
WHERE TRIGGER_SCHEMA = 'shop';

-- Drop trigger
DROP TRIGGER IF EXISTS before_user_delete;
```

---

### Trigger Limitations

| Limitation | Description |
|------|------|
| Max 6 triggers per table | One for each event type (BEFORE/AFTER x INSERT/UPDATE/DELETE) |
| Cannot be recursive | Triggers cannot call themselves |
| Implicit commits | Cannot use transaction control statements in triggers (START TRANSACTION, COMMIT, ROLLBACK) |
| Performance impact | Each DML operation adds extra overhead |
| Hard to debug | Trigger behavior is implicit, can be confusing |

### When to Use Triggers?

| ✅ Suitable | ❌ Not Suitable |
|---------|----------|
| Audit logging | Complex business logic |
| Simple data validation | Cross-service calls |
| Automated derived data maintenance | High-frequency write scenarios (amplifies write overhead) |
| Simple data synchronization | High-concurrency inventory deduction |

**Rule of thumb**: Avoid triggers where possible. Many things triggers can do are more controllable and maintainable in the application layer.

---

## 2. Event

An event is a **scheduled task** in MySQL that automatically executes SQL according to a specified schedule.

### Enabling the Event Scheduler

```sql
-- Check if enabled
SHOW VARIABLES LIKE 'event_scheduler';

-- Enable (temporary)
SET GLOBAL event_scheduler = ON;

-- Permanently enable in config (my.cnf)
-- event_scheduler = ON
```

### Creating Events

```sql
DELIMITER //

CREATE EVENT daily_cleanup
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 03:00:00'
DO
BEGIN
    -- Delete logs older than 90 days
    DELETE FROM operation_logs WHERE created_at < NOW() - INTERVAL 90 DAY;

    -- Mark expired unpaid orders as cancelled
    UPDATE orders SET status = 4
    WHERE status = 0 AND created_at < NOW() - INTERVAL 1 DAY;
END //

DELIMITER ;
```

### Event Schedule Configuration

```sql
-- One-time event
CREATE EVENT one_time_cleanup
ON SCHEDULE AT '2025-02-01 02:00:00'
DO
    DELETE FROM temp_data WHERE created_at < NOW();

-- Recurring event
CREATE EVENT hourly_stats
ON SCHEDULE EVERY 1 HOUR
STARTS '2025-01-01 00:00:00'
ENDS '2025-12-31 23:59:59'
DO
    INSERT INTO hourly_orders (hour, order_count, total_amount)
    SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:00:00'), COUNT(*), SUM(total_amount)
    FROM orders
    WHERE created_at >= NOW() - INTERVAL 1 HOUR;

-- Every 5 minutes
CREATE EVENT every_5_minutes
ON SCHEDULE EVERY 5 MINUTE
DO
    CALL process_pending_jobs();
```

### Event Management

```sql
-- View events
SHOW EVENTS;
SHOW EVENTS FROM shop;

-- View event definition
SHOW CREATE EVENT daily_cleanup;

-- Modify event
ALTER EVENT daily_cleanup
ON SCHEDULE EVERY 6 HOUR;

-- Enable/disable
ALTER EVENT daily_cleanup ENABLE;
ALTER EVENT daily_cleanup DISABLE;

-- Drop
DROP EVENT IF EXISTS daily_cleanup;
```

### Event Status Checks

```sql
-- View event execution history
SELECT * FROM information_schema.EVENTS
WHERE EVENT_SCHEMA = 'shop';

-- View process list
SHOW PROCESSLIST;
-- You will see an event_scheduler thread
```

---

## 3. Practical Scenarios

### Scenario 1: Automatic Expiry Cleanup

```sql
CREATE EVENT clean_expired_sessions
ON SCHEDULE EVERY 30 MINUTE
DO
BEGIN
    -- Clean expired sessions
    DELETE FROM sessions WHERE expires_at < NOW();

    -- Clean unverified users (email not verified within 24 hours)
    UPDATE users SET status = -1
    WHERE status = 0 AND email_verified_at IS NULL
      AND created_at < NOW() - INTERVAL 24 HOUR;

    -- Restore inventory for cancelled orders
    UPDATE products p
    JOIN order_items oi ON oi.product_id = p.id
    JOIN orders o ON o.id = oi.order_id
    SET p.stock = p.stock + oi.quantity
    WHERE o.status = 4 AND o.updated_at < NOW() - INTERVAL 1 HOUR;
END;
```

### Scenario 2: Auto-generate Daily Report

```sql
CREATE EVENT generate_daily_summary
ON SCHEDULE EVERY 1 DAY
STARTS '2025-01-01 23:55:00'
DO
BEGIN
    INSERT INTO daily_summary (summary_date, new_users, new_orders, revenue)
    SELECT
        CURDATE() - INTERVAL 1 DAY,
        (SELECT COUNT(*) FROM users WHERE DATE(created_at) = CURDATE() - INTERVAL 1 DAY),
        (SELECT COUNT(*) FROM orders WHERE DATE(created_at) = CURDATE() - INTERVAL 1 DAY),
        (SELECT COALESCE(SUM(total_amount), 0) FROM orders
         WHERE DATE(paid_at) = CURDATE() - INTERVAL 1 DAY AND status = 1);
END;
```

### Scenario 3: Periodic Data Archiving

```sql
CREATE EVENT archive_old_orders
ON SCHEDULE EVERY 1 MONTH
STARTS '2025-02-01 04:00:00'
DO
BEGIN
    -- Move orders older than one year to archive table
    INSERT INTO orders_archive
    SELECT * FROM orders
    WHERE created_at < NOW() - INTERVAL 1 YEAR
      AND NOT EXISTS (SELECT 1 FROM orders_archive oa WHERE oa.id = orders.id);

    -- Delete archived original data
    DELETE FROM orders
    WHERE created_at < NOW() - INTERVAL 1 YEAR;
END;
```

---

## 4. Event vs System Cron Jobs

| Comparison | MySQL Event | System cron |
|------|------------|-----------|
| Dependency | MySQL instance must be running | System crond |
| Precision | Precise to the second | Precise to the minute |
| Management | SQL operations, convenient | Requires SSH |
| Backup | Included in DB backup | Requires separate backup |
| Resources | Uses DB connections | Independent process |

**Recommendation**:
- Simple database maintenance tasks → MySQL Event
- Complex cross-system tasks → System cron (e.g., calling external APIs, sending emails)

---

## Summary

Triggers and events are MySQL's automation capabilities. Triggers are suitable for simple data integrity checks and auditing, while events are used for in-database scheduled tasks. Control the scope of trigger usage to avoid problems caused by implicit logic.

---

