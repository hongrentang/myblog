---
title: "MySQL from Beginner to Pro · Part 3: Data Types and Constraints"
date: 2025-01-23
weight: 3
draft: false
tags: ["mysql"]
---

## 1. Numeric Types

### Integer Types

| Type | Bytes | Range (Signed) | Range (Unsigned) |
|------|---------|-------------|-------------|
| `TINYINT` | 1 | -128 ~ 127 | 0 ~ 255 |
| `SMALLINT` | 2 | -32768 ~ 32767 | 0 ~ 65535 |
| `MEDIUMINT` | 3 | -8388608 ~ 8388607 | 0 ~ 16777215 |
| `INT` | 4 | -2^31 ~ 2^31-1 | 0 ~ 2^32-1 |
| `BIGINT` | 8 | -2^63 ~ 2^63-1 | 0 ~ 2^64-1 |

```sql
-- Unsigned example
age TINYINT UNSIGNED DEFAULT 0         -- age >= 0

-- Boolean values typically use TINYINT(1)
is_active TINYINT(1) NOT NULL DEFAULT 1

-- Primary key recommended: BIGINT UNSIGNED AUTO_INCREMENT
id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY
```

### Floating Point and Fixed Point

| Type | Description | Use Case |
|------|------|---------|
| `FLOAT` | 4 bytes, ~7 digits precision | Scientific calculations, error tolerance |
| `DOUBLE` | 8 bytes, ~15 digits precision | Scientific calculations |
| `DECIMAL(M,D)` | Exact decimal | **Currency, prices** |

```sql
price DECIMAL(10,2) NOT NULL  -- Total 10 digits, 2 decimal places, range 0 ~ 99999999.99

-- ❌ Do not use FLOAT/DOUBLE for currency
price FLOAT  -- Might store as 9.99 but read back as 9.990000000000002
```

**Key Rule**: For scenarios involving money or accounts, **must** use `DECIMAL`.

---

## 2. String Types

| Type | Description | Max Length | Features |
|------|------|---------|------|
| `CHAR(M)` | Fixed-length string | 255 chars | Fixed length, fast, suitable for fixed-length fields |
| `VARCHAR(M)` | Variable-length string | 65535 bytes ≈ 16383 UTF-8 chars | Flexible, occupies actual length + 1~2 bytes |
| `TINYTEXT` | Short text | 255 bytes | — |
| `TEXT` | Long text | 65535 bytes | — |
| `MEDIUMTEXT` | Medium text | 16MB | — |
| `LONGTEXT` | Very long text | 4GB | — |
| `BLOB` | Binary data | Similar to TEXT | Images, files, etc. |

### CHAR vs VARCHAR

```sql
-- CHAR: Fixed length, padded with spaces if shorter
code CHAR(6)  -- Storing 'ABC' occupies 6 bytes

-- VARCHAR: Variable length, 1~2 bytes overhead
name VARCHAR(50)  -- Storing 'ABC' occupies 3 + 1 = 4 bytes
```

| Scenario | Recommended Type |
|------|---------|
| Fixed codes (phone numbers, postal codes) | `CHAR` |
| Variable length (names, titles, addresses) | `VARCHAR` |
| Short unique codes (order numbers, ID numbers) | `CHAR` |
| Large text (articles, descriptions) | `TEXT` |

### VARCHAR Length Selection

MySQL has limits on VARCHAR index length (InnoDB max 767 bytes):

```sql
-- If adding an index to a VARCHAR field, watch the length
name VARCHAR(255)   -- Index prefix needs 255 × 3 = 765 bytes, close to limit
name VARCHAR(100)   -- Recommended
```

**Common Practices**:
- Name fields: `VARCHAR(50)` ~ `VARCHAR(100)`
- Email: `VARCHAR(100)`
- Phone number: `CHAR(11)`
- URL: `VARCHAR(500)`
- Uncertain length: `VARCHAR(255)` is a common upper limit

---

## 3. Date and Time Types

| Type | Size | Range | Precision | Recommendation |
|------|------|------|------|------|
| `DATE` | 3 bytes | 1000-01-01 ~ 9999-12-31 | Day | Birthdays, dates |
| `DATETIME` | 5 bytes | Same as above | Second/microsecond | **Most commonly used** |
| `TIMESTAMP` | 4 bytes | 1970-01-01 ~ 2038-01-19 | Second/microsecond | Timezone-dependent |
| `TIME` | 3 bytes | -838:59:59 ~ 838:59:59 | Second/microsecond | Time intervals |
| `YEAR` | 1 byte | 1901 ~ 2155 | Year | Years |

### DATETIME vs TIMESTAMP

```sql
-- DATETIME: Timezone-independent, large range
created_at DATETIME DEFAULT CURRENT_TIMESTAMP

-- TIMESTAMP: Auto-converted to UTC for storage, converted to current timezone on query
updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

| Comparison | DATETIME | TIMESTAMP |
|------|----------|-----------|
| Storage | 5 bytes | 4 bytes |
| Timezone | Not timezone-aware | Affected by timezone |
| Range | 1000 ~ 9999 | 1970 ~ 2038 |
| Index efficiency | Good | Good |

**Recommendation**: Generally prefer `DATETIME` to avoid the year 2038 problem.

### Date Functions

```sql
-- Get current time
SELECT NOW(), CURDATE(), CURTIME();

-- Date formatting
SELECT DATE_FORMAT(created_at, '%Y-%m-%d %H:%i:%s') FROM users;

-- Date arithmetic
SELECT DATE_ADD(created_at, INTERVAL 7 DAY) FROM users;
SELECT DATEDIFF(NOW(), created_at) FROM users;

-- Extract parts
SELECT YEAR(created_at), MONTH(created_at), DAY(created_at) FROM users;
```

---

## 4. Other Types

### JSON Type (MySQL 5.7+)

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    attributes JSON
);

INSERT INTO products VALUES (1, '手机', '{"color": "黑色", "storage": "256GB", "ram": "8GB"}');

-- Query JSON fields
SELECT name, JSON_EXTRACT(attributes, '$.color') AS color FROM products;

-- MySQL 8.0+ shorthand
SELECT name, attributes->>'$.color' AS color FROM products;

-- JSON as virtual column with index
ALTER TABLE products ADD COLUMN color VARCHAR(20) GENERATED ALWAYS AS (attributes->>'$.color') STORED;
CREATE INDEX idx_color ON products(color);
```

### ENUM Type

```sql
status ENUM('pending', 'paid', 'shipped', 'completed')

-- Pros: Good readability
-- Cons: Modifying enum values requires ALTER TABLE, inflexible
```

**Recommendation**: If enum values are few and rarely change (e.g., gender), use ENUM. Otherwise, use `TINYINT` + comments.

### SET Type

```sql
permissions SET('read', 'write', 'delete', 'admin')
```

Not commonly used. Use a related table instead.

---

## 5. Constraint Types

### NOT NULL

```sql
name VARCHAR(50) NOT NULL
```

**Design Principle**: Use `NOT NULL` where possible; use `DEFAULT` instead of NULL for uncertain fields.

### UNIQUE

```sql
-- Single field unique
email VARCHAR(100) UNIQUE NOT NULL

-- Composite unique: one user can only post one comment per day
UNIQUE KEY uk_user_date (user_id, DATE(created_at))
```

### PRIMARY KEY

```sql
-- Single column primary key (auto-increment BIGINT recommended)
id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY

-- Composite primary key
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

### DEFAULT

```sql
status TINYINT NOT NULL DEFAULT 0
created_at DATETIME DEFAULT CURRENT_TIMESTAMP
updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

### CHECK (Only truly effective in MySQL 8.0+)

```sql
age INT CHECK (age >= 0 AND age <= 150)
price DECIMAL(10,2) CHECK (price >= 0)
status VARCHAR(10) CHECK (status IN ('active', 'inactive'))
```

> Note: CHECK constraints are only truly effective in MySQL 8.0.16+. In earlier versions, the syntax is accepted but ignored.

### FOREIGN KEY

```sql
order_id INT NOT NULL,
CONSTRAINT fk_order_user FOREIGN KEY (user_id) REFERENCES users(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE
```

Foreign key constraint actions:

| Action | Description |
|------|------|
| `CASCADE` | Cascading delete/update |
| `SET NULL` | Set child table to NULL when parent row is deleted |
| `RESTRICT` | Prevent deletion (default) |
| `NO ACTION` | Same as RESTRICT |

---

## 6. Field Design Principles

```
id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY  -- Primary key
user_id     INT UNSIGNED NOT NULL                       -- Foreign key
xxx_name    VARCHAR(50) NOT NULL                        -- Name
xxx_status  TINYINT NOT NULL DEFAULT 0                  -- Status
xxx_at      DATETIME DEFAULT CURRENT_TIMESTAMP           -- Time field
```

### 1. Prefer NOT NULL

```sql
-- ❌
email VARCHAR(100)

-- ✅
email VARCHAR(100) NOT NULL
-- If truly unknown, set DEFAULT ''
```

### 2. Use TINYINT instead of ENUM

```sql
-- ❌
status ENUM('pending','paid','cancelled')

-- ✅ (Easier to extend)
status TINYINT NOT NULL DEFAULT 0 COMMENT '0=pending 1=paid 2=cancelled'
```

### 3. Consistent Time Fields

```sql
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

---

## Summary

Data types and constraints are the foundation of table design. Choosing the right data type not only saves space but also improves query performance.

---

