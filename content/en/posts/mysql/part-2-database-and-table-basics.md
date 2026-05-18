---
title: "MySQL from Beginner to Pro · Part 2: Database and Table Basics"
date: 2025-01-22
weight: 2
draft: false
tags: ["mysql"]
---

## Database Operations

### Creating a Database

```sql
-- Basic creation
CREATE DATABASE shop;

-- Specify character set and collation
CREATE DATABASE shop
  DEFAULT CHARACTER SET utf8mb4
  DEFAULT COLLATE utf8mb4_unicode_ci;

-- Create if not exists (avoid errors from duplicate creation)
CREATE DATABASE IF NOT EXISTS shop;
```

**Character Set Recommendation**: Always use `utf8mb4` as it supports the full Unicode character range (including emoji). `utf8` in MySQL only supports up to 3-byte characters.

### Viewing and Selecting Databases

```sql
-- View all databases
SHOW DATABASES;

-- Show current database
SELECT DATABASE();

-- Switch database
USE shop;
```

### Modifying a Database

```sql
ALTER DATABASE shop CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Deleting a Database

```sql
-- ⚠️ Deletion is irreversible
DROP DATABASE shop;
DROP DATABASE IF EXISTS shop;
```

---

## Table Operations

### Naming Conventions

| Item | Recommendation |
|------|------|
| Database name | Lowercase English words with underscores, e.g., `shop_order` |
| Table name | Plural nouns, e.g., `users`, `orders`, `products` |
| Column name | Lowercase snake_case, e.g., `user_name`, `created_at` |
| Primary key | Use `id` consistently |
| Foreign key | Related table name + `_id`, e.g., `user_id` |

### Creating a Table

```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    age INT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- View table structure
DESC users;

-- View CREATE TABLE statement
SHOW CREATE TABLE users;
```

### Complete Table Creation Example

```sql
CREATE TABLE orders (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_no    VARCHAR(32) NOT NULL COMMENT 'Order number',
    user_id     INT NOT NULL COMMENT 'User ID',
    total_amount DECIMAL(10,2) NOT NULL DEFAULT 0.00 COMMENT 'Total amount',
    status      TINYINT NOT NULL DEFAULT 0 COMMENT 'Status:0=pending 1=paid 2=shipped 3=completed',
    paid_at     DATETIME COMMENT 'Payment time',
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY uk_order_no (order_no),
    KEY idx_user_id (user_id),
    KEY idx_status (status),
    KEY idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Orders table';
```

---

## Altering Table Structure

### Adding Columns

```sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20) AFTER email;
ALTER TABLE users ADD COLUMN address VARCHAR(200) DEFAULT '';
ALTER TABLE users ADD COLUMN gender TINYINT AFTER name;
```

`AFTER` specifies the position; without `AFTER`, the column is added at the end by default.

### Modifying Columns

```sql
-- Modify column type
ALTER TABLE users MODIFY COLUMN phone VARCHAR(30);

-- Modify column name and type
ALTER TABLE users CHANGE COLUMN phone mobile VARCHAR(20);

-- Set default value
ALTER TABLE users ALTER COLUMN age SET DEFAULT 18;

-- Remove default value
ALTER TABLE users ALTER COLUMN age DROP DEFAULT;
```

### Dropping Columns

```sql
ALTER TABLE users DROP COLUMN address;
```

### Renaming Columns

```sql
ALTER TABLE users RENAME COLUMN mobile TO phone;
```

---

## Index Operations

### Normal Index

```sql
-- Create index
CREATE INDEX idx_email ON users(email);

-- Create unique index
CREATE UNIQUE INDEX idx_email_unique ON users(email);

-- Composite index
CREATE INDEX idx_name_age ON users(name, age);

-- View indexes
SHOW INDEX FROM users;

-- Drop index
DROP INDEX idx_email ON users;
```

### Creating Indexes with ALTER TABLE

```sql
ALTER TABLE users ADD INDEX idx_name (name);
ALTER TABLE users ADD UNIQUE INDEX idx_email (email);
ALTER TABLE users ADD FULLTEXT INDEX idx_content (content);  -- Full-text index
```

---

## Constraint Operations

```sql
-- Add primary key
ALTER TABLE users ADD PRIMARY KEY (id);

-- Add unique constraint
ALTER TABLE users ADD UNIQUE KEY uk_email (email);

-- Add foreign key constraint
ALTER TABLE orders ADD CONSTRAINT fk_user
    FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE ON UPDATE CASCADE;

-- Drop foreign key
ALTER TABLE orders DROP FOREIGN KEY fk_user;
```

---

## Renaming Tables

```sql
RENAME TABLE users TO members;
-- or
ALTER TABLE users RENAME TO members;
```

---

## Truncating and Dropping Tables

```sql
-- Clear table data (non-rollbackable, auto-increment resets)
TRUNCATE TABLE temp_logs;

-- Drop table structure and data
DROP TABLE temp_logs;
DROP TABLE IF EXISTS temp_logs;
```

| Operation | Speed | Rollback | Auto-increment | DDL/DML |
|------|------|------|--------|---------|
| `DELETE FROM t` | Slow (row-by-row) | Rollbackable | Unchanged | DML |
| `TRUNCATE t` | Fast | Not rollbackable | Reset | DDL |
| `DROP TABLE t` | Fast | Not rollbackable | — | DDL |

---

## Useful Information Queries

```sql
-- View all tables
SHOW TABLES;
SHOW TABLES FROM mysql;

-- View table structure
DESC users;
SHOW COLUMNS FROM users;

-- View CREATE TABLE statement
SHOW CREATE TABLE users;

-- View table size
SELECT
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema = 'shop' AND table_name = 'users';

-- View all table sizes
SELECT
    table_name,
    ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
FROM information_schema.tables
WHERE table_schema = 'shop'
ORDER BY size_mb DESC;
```

---

## Best Practices

### 1. Always Specify Character Set

```sql
CREATE DATABASE db CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE TABLE t (...) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 2. Column Order in Tables

```sql
CREATE TABLE good_example (
    id          BIGINT UNSIGNED AUTO_INCREMENT,   -- Primary key
    user_id     INT UNSIGNED NOT NULL,             -- Foreign key
    name        VARCHAR(50) NOT NULL,              -- Business field
    status      TINYINT NOT NULL DEFAULT 0,        -- Status
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,-- Time fields at the end
    updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
    PRIMARY KEY (id)
);
```

### 3. Column Comments

Add comments to every field for team collaboration:

```sql
CREATE TABLE products (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY COMMENT 'Product ID',
    name        VARCHAR(200) NOT NULL COMMENT 'Product name',
    price       DECIMAL(10,2) NOT NULL COMMENT 'Unit price (yuan)',
    stock       INT NOT NULL DEFAULT 0 COMMENT 'Stock quantity',
    status      TINYINT NOT NULL DEFAULT 1 COMMENT 'Status:1=active 0=inactive',
    created_at  DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT 'Creation time'
) COMMENT='Products table';
```

### 4. Avoid Reserved Words

Do not use MySQL reserved words as table or column names. If necessary, use backticks:

```sql
-- ❌ Not recommended
CREATE TABLE order (...)  -- order is a reserved word

-- ✅ Correct
CREATE TABLE `order` (...)

-- ✅ Recommended
CREATE TABLE orders (...)
```

---

## Summary

This article covered database and table CRUD operations in detail, as well as index and constraint design.

---

