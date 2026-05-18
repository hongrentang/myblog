---
title: "MySQL from Beginner to Pro · Part 4: CRUD Operations"
date: 2025-01-24
weight: 4
draft: false
tags: ["mysql"]
---

CRUD is the most fundamental database operation: **Create**, **Read**, **Update**, **Delete**.

---

## 1. INSERT -- Inserting Data

### Basic Syntax

```sql
-- Insert single row (specified fields)
INSERT INTO users (name, email, age)
VALUES ('张三', 'zhangsan@example.com', 25);

-- Insert single row (all fields, not recommended)
INSERT INTO users VALUES (1, '张三', 'zhangsan@example.com', 25, NOW(), NOW());

-- Insert multiple rows
INSERT INTO users (name, email, age) VALUES
    ('李四', 'lisi@example.com', 30),
    ('王五', 'wangwu@example.com', 28),
    ('赵六', 'zhaoliu@example.com', 22);
```

### Advanced Usage

```sql
-- INSERT ... SELECT (query from another table and insert)
INSERT INTO vip_users (name, email)
SELECT name, email FROM users WHERE age > 25;

-- INSERT IGNORE (ignore on duplicate, no error)
INSERT IGNORE INTO users (id, name, email) VALUES (1, '张三', 'zhangsan@example.com');

-- REPLACE (replace if exists, insert if not)
REPLACE INTO users (id, name, email) VALUES (1, '张三2', 'zhangsan2@example.com');

-- ON DUPLICATE KEY UPDATE (update on duplicate)
INSERT INTO users (id, name, email) VALUES (1, '张三', 'zhangsan@example.com')
ON DUPLICATE KEY UPDATE name = VALUES(name), email = VALUES(email);
```

### INSERT Performance Optimization

```sql
-- For large batch inserts, commit in segments
INSERT INTO logs (message) VALUES ('msg1'), ('msg2'), ...;  -- 500~1000 at a time

-- Temporarily disable unique checks (re-enable after import)
SET UNIQUE_CHECKS = 0;
-- Execute many INSERTs ...
SET UNIQUE_CHECKS = 1;
```

---

## 2. SELECT -- Querying Data

### Basic Queries

```sql
-- Query all fields (avoid in production if possible)
SELECT * FROM users;

-- Query specific fields
SELECT id, name, email FROM users;

-- Aliases
SELECT name AS 姓名, email AS 邮箱 FROM users;

-- Deduplicate
SELECT DISTINCT status FROM orders;

-- Constant columns
SELECT name, 'VIP' AS user_type FROM users;
```

### Conditional Queries

```sql
-- Comparison operators
SELECT * FROM users WHERE age >= 18;
SELECT * FROM orders WHERE status != 0;

-- Range
SELECT * FROM users WHERE age BETWEEN 18 AND 35;
SELECT * FROM orders WHERE total_amount > 100;

-- IN list
SELECT * FROM users WHERE id IN (1, 3, 5, 7);

-- Fuzzy search
SELECT * FROM users WHERE name LIKE '张%';    -- Starts with "张"
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- Gmail emails

-- NULL checks
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;
```

---

## 3. UPDATE -- Updating Data

### Basic Syntax

```sql
-- Update single field
UPDATE users SET age = 26 WHERE id = 1;

-- Update multiple fields
UPDATE users SET age = 26, phone = '13800138000' WHERE id = 1;

-- Without WHERE, updates entire table!
UPDATE users SET status = 1;  -- ⚠️ Full table update
```

### Conditional Updates

```sql
-- Update matching data
UPDATE orders SET status = 1, paid_at = NOW()
WHERE status = 0 AND created_at < DATE_SUB(NOW(), INTERVAL 30 MINUTE);

-- Use CASE WHEN for batch update
UPDATE users SET age = CASE id
    WHEN 1 THEN 20
    WHEN 2 THEN 25
    WHEN 3 THEN 30
END
WHERE id IN (1, 2, 3);
```

### UPDATE Caveats

```sql
-- 1. Use existing field values when updating
UPDATE products SET stock = stock - 1 WHERE id = 1;
-- Safer and more efficient than SELECT then UPDATE (atomic operation)

-- 2. Use LIMIT to restrict affected rows
UPDATE users SET status = 1 WHERE status = 0 LIMIT 100;

-- 3. UPDATE with JOIN
UPDATE orders o
JOIN users u ON o.user_id = u.id
SET o.status = 1
WHERE u.email = 'zhangsan@example.com';
```

---

## 4. DELETE -- Deleting Data

### Basic Syntax

```sql
-- Delete specific rows
DELETE FROM users WHERE id = 1;

-- Delete all rows (but auto-increment is not reset)
DELETE FROM temp_logs;

-- Safe delete with LIMIT
DELETE FROM logs WHERE created_at < '2024-01-01' LIMIT 1000;
```

### Safe Delete Pattern

```sql
-- First confirm the data to be deleted
SELECT * FROM users WHERE email = 'bad@example.com';

-- Delete after confirmation
DELETE FROM users WHERE email = 'bad@example.com';

-- Or use soft delete
ALTER TABLE users ADD COLUMN deleted_at DATETIME DEFAULT NULL;
-- When "deleting":
UPDATE users SET deleted_at = NOW() WHERE id = 1;
-- When querying, filter out deleted:
SELECT * FROM users WHERE deleted_at IS NULL;
```

**Soft delete (logical deletion)** is recommended -- data is not lost and can be recovered.

---

## 5. Pagination

```sql
-- LIMIT offset, count
SELECT * FROM users ORDER BY id LIMIT 0, 20;   -- Page 1
SELECT * FROM users ORDER BY id LIMIT 20, 20;  -- Page 2
SELECT * FROM users ORDER BY id LIMIT 40, 20;  -- Page 3

-- LIMIT count OFFSET offset (equivalent syntax)
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 40;
```

### Deep Pagination Optimization

```sql
-- ❌ Traditional deep pagination (slower as page number increases)
SELECT * FROM users ORDER BY id LIMIT 100000, 20;

-- ✅ Cursor-based pagination (recommended)
SELECT * FROM users WHERE id > 100000 ORDER BY id LIMIT 20;

-- ✅ Subquery optimization
SELECT * FROM users
WHERE id >= (SELECT id FROM users ORDER BY id LIMIT 100000, 1)
ORDER BY id LIMIT 20;
```

---

## 6. Aggregate Queries

```sql
-- Common aggregate functions
SELECT
    COUNT(*) AS total_users,
    AVG(age) AS avg_age,
    MAX(age) AS max_age,
    MIN(age) AS min_age,
    SUM(balance) AS total_balance
FROM users;

-- Group statistics
SELECT status, COUNT(*) AS cnt, SUM(total_amount) AS total
FROM orders
GROUP BY status;

-- HAVING to filter aggregate results
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING order_count > 5;

-- GROUP_CONCAT to concatenate
SELECT user_id, GROUP_CONCAT(order_no SEPARATOR ',') AS orders
FROM orders
GROUP BY user_id;
```

---

## 7. Sorting

```sql
-- Ascending
SELECT * FROM products ORDER BY price ASC;

-- Descending
SELECT * FROM products ORDER BY price DESC;

-- Multi-field sorting
SELECT * FROM products ORDER BY category_id, price DESC;

-- Sort by expression
SELECT * FROM products ORDER BY price * discount ASC;

-- Sort by field value (custom sorting)
SELECT * FROM users ORDER BY FIELD(status, 'active', 'pending', 'disabled');
```

---

## 8. Complete Query Order

```sql
SELECT DISTINCT columns              -- 5. Select output columns
FROM table                           -- 1. Determine data source
JOIN other_table ON condition        -- 2. Multi-table join
WHERE condition                      -- 3. Row filtering
GROUP BY columns                     -- 4. Grouping
HAVING aggregate_condition           -- 4.5 Post-group filtering
ORDER BY columns                     -- 6. Sorting
LIMIT count OFFSET offset;           -- 7. Pagination
```

Understanding this execution order is very helpful for writing correct SQL.

---

## Practice Exercise

Using an order system as an example, apply CRUD comprehensively:

```sql
-- 1. Register user
INSERT INTO users (name, email, phone) VALUES ('张三', 'zs@test.com', '13800001111');

-- 2. User places order
INSERT INTO orders (order_no, user_id, total_amount, status)
VALUES ('20250115001', '1', 299.00, 0);

-- 3. Query user orders
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.email = 'zs@test.com'
ORDER BY o.created_at DESC;

-- 4. Pay for order
UPDATE orders SET status = 1, paid_at = NOW()
WHERE order_no = '20250115001';

-- 5. Cancel expired unpaid orders
UPDATE orders SET status = 4
WHERE status = 0 AND created_at < NOW() - INTERVAL 1 DAY;

-- 6. User spending ranking
SELECT u.name, COUNT(o.id) AS order_count, SUM(o.total_amount) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 1
GROUP BY u.id, u.name
ORDER BY total_spent DESC
LIMIT 10;
```

---

## Summary

This article covered the complete usage of CRUD, including inserts, queries, updates, deletes, pagination, and aggregation. Mastering these operations is fundamental to database development.

---

