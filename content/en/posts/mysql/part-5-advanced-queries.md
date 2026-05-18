---
title: "MySQL from Beginner to Pro · Part 5: Advanced Queries"
date: 2025-01-25
weight: 5
draft: false
tags: ["mysql"]
---

## 1. WHERE Conditions in Detail

### Comparison Operators

```sql
-- Basic comparisons
WHERE age = 18
WHERE age != 18
WHERE age <> 18      -- Inequality operator, same as !=
WHERE age > 18
WHERE age >= 18
WHERE age BETWEEN 18 AND 35

-- String comparison
WHERE name = '张三'
WHERE name > '李'    -- Lexicographic comparison (UTF-8 encoding order)
```

### Logical Operators

```sql
-- AND
WHERE age >= 18 AND status = 1

-- OR
WHERE city = '北京' OR city = '上海'

-- NOT or <>
WHERE NOT status = 0
WHERE status <> 0

-- Precedence: NOT > AND > OR
-- Use parentheses to clarify precedence:
WHERE (status = 1 OR level = 'vip') AND age >= 18
```

### IN and NOT IN

```sql
-- IN list
WHERE city IN ('北京', '上海', '广州')
WHERE city NOT IN ('北京', '上海')

-- IN subquery (watch performance, EXISTS is better for large datasets)
WHERE user_id IN (SELECT id FROM vip_users)
```

### LIKE Fuzzy Search

```sql
WHERE name LIKE '张%'         -- Starts with "张"
WHERE name LIKE '%张'         -- Ends with "张"
WHERE name LIKE '%张%'        -- Contains "张"
WHERE name LIKE '张_'         -- "张" + any single character
WHERE name LIKE '张__'        -- "张" + any two characters
```

### NULL Checks

```sql
WHERE phone IS NULL           -- Is NULL
WHERE phone IS NOT NULL       -- Is not NULL

-- ⚠️ Incorrect:
WHERE phone = NULL     -- Never true! NULL cannot be compared with =
WHERE phone != NULL    -- Also never true
```

### EXISTS

```sql
-- Query users who have orders
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- Query users without orders
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

Choosing between `EXISTS` and `IN`:
- Small subquery result set → `IN` may be faster
- Small outer table, large subquery → `EXISTS` is better
- If `JOIN` can be used, prefer `JOIN`

---

## 2. ORDER BY Sorting

### Single Field Sorting

```sql
SELECT * FROM products ORDER BY price;          -- Ascending (default)
SELECT * FROM products ORDER BY price ASC;      -- Ascending (explicit)
SELECT * FROM products ORDER BY price DESC;     -- Descending
```

### Multi-Field Sorting

```sql
-- Sort by category ascending, then by price descending
SELECT * FROM products ORDER BY category_id ASC, price DESC;
```

### Expression Sorting

```sql
-- Sort by discounted price
SELECT *, price * discount AS final_price
FROM products
ORDER BY final_price;

-- Sort by length
SELECT * FROM articles ORDER BY LENGTH(content) DESC;

-- Random sort (extremely slow on large datasets, use sparingly)
SELECT * FROM users ORDER BY RAND() LIMIT 10;
```

### Sorting and Indexes

```sql
-- If the sort field has an index, sorting is fast
-- If a join involves sorting, sorting typically occurs after the join
-- Create indexes on frequently sorted fields like amounts and timestamps
```

---

## 3. GROUP BY Aggregation

### Basic Grouping

```sql
-- Group and count by status
SELECT status, COUNT(*) AS cnt
FROM orders
GROUP BY status;

-- Group by year
SELECT YEAR(created_at) AS year, SUM(total_amount) AS total
FROM orders
GROUP BY YEAR(created_at);
```

### Multi-Field Grouping

```sql
-- Group by year and month
SELECT
    YEAR(created_at) AS year,
    MONTH(created_at) AS month,
    COUNT(*) AS order_count,
    SUM(total_amount) AS revenue
FROM orders
GROUP BY YEAR(created_at), MONTH(created_at)
ORDER BY year DESC, month DESC;
```

### HAVING Filter

```sql
-- WHERE filters before grouping, HAVING filters after grouping
SELECT user_id, COUNT(*) AS order_count, SUM(total_amount) AS total
FROM orders
WHERE status = 1               -- Only count paid orders
GROUP BY user_id
HAVING order_count >= 5        -- Only show users with >= 5 orders
   AND total > 1000;            -- And total amount > 1000
```

### GROUP_CONCAT

```sql
-- Concatenate values within a group into a string
SELECT
    user_id,
    GROUP_CONCAT(order_no ORDER BY created_at DESC SEPARATOR ', ') AS orders
FROM orders
GROUP BY user_id;
```

---

## 4. LIMIT Pagination

### Basic Pagination

```sql
-- LIMIT offset, count
SELECT * FROM users ORDER BY id LIMIT 0, 20;     -- Page 1 (rows 1-20)
SELECT * FROM users ORDER BY id LIMIT 20, 20;    -- Page 2 (rows 21-40)
SELECT * FROM users ORDER BY id LIMIT 40, 20;    -- Page 3 (rows 41-60)

-- Alternative syntax
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 40;
```

### Deep Pagination Optimization

```sql
-- ❌ When offset is very large, MySQL needs to scan and discard the first 100000 rows
SELECT * FROM users ORDER BY id LIMIT 100000, 20;

-- ✅ Solution 1: Cursor-based (recommended)
SELECT * FROM users WHERE id > 100000 ORDER BY id LIMIT 20;

-- ✅ Solution 2: Subquery optimization
SELECT * FROM users
WHERE id >= (SELECT id FROM users ORDER BY id LIMIT 100000, 1)
ORDER BY id LIMIT 20;

-- ✅ Solution 3: JOIN optimization
SELECT u.* FROM users u
JOIN (SELECT id FROM users ORDER BY id LIMIT 100000, 20) AS tmp
ON u.id = tmp.id;
```

### Top-N Queries

```sql
-- Highest priced product in each category
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS rn
    FROM products
) t WHERE rn = 1;
```

---

## 5. Conditional Expressions

### CASE WHEN

```sql
SELECT
    name,
    CASE
        WHEN age < 18 THEN '少年'
        WHEN age < 35 THEN '青年'
        WHEN age < 60 THEN '中年'
        ELSE '老年'
    END AS age_group
FROM users;

-- Simple CASE (equality check)
SELECT
    status,
    CASE status
        WHEN 0 THEN '待支付'
        WHEN 1 THEN '已支付'
        WHEN 2 THEN '已发货'
        WHEN 3 THEN '已完成'
        ELSE '未知'
    END AS status_text
FROM orders;
```

### IF Function

```sql
SELECT name, IF(age >= 18, '成年', '未成年') AS adult FROM users;

-- IFNULL (NULL handling)
SELECT name, IFNULL(phone, '未填写') AS phone FROM users;

-- COALESCE (return first non-NULL value)
SELECT COALESCE(phone, email, '无联系方式') AS contact FROM users;
```

---

## 6. Advanced Aggregate Functions

```sql
-- Common aggregates
COUNT(*)        -- Total rows
COUNT(col)      -- Non-NULL rows
COUNT(DISTINCT col)  -- Distinct count

SUM(col)        -- Sum
AVG(col)        -- Average

MAX(col)        -- Maximum
MIN(col)        -- Minimum

GROUP_CONCAT(col)  -- String concatenation

-- Summary statistics (WITH ROLLUP)
SELECT
    IFNULL(status, '合计') AS status,
    COUNT(*) AS cnt,
    SUM(total_amount) AS total
FROM orders
GROUP BY status WITH ROLLUP;
```

---

## 7. Window Functions (MySQL 8.0+)

### Ranking Functions

```sql
SELECT
    name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS ranking,         -- No ties
    RANK() OVER (ORDER BY salary DESC) AS rank_with_gap,         -- Ties with gaps
    DENSE_RANK() OVER (ORDER BY salary DESC) AS rank_no_gap      -- Ties without gaps
FROM employees;
```

### Aggregate Window

```sql
SELECT
    DATE(created_at) AS day,
    SUM(amount) AS daily_total,
    SUM(SUM(amount)) OVER (ORDER BY DATE(created_at)) AS running_total
FROM orders
GROUP BY DATE(created_at);
```

### Partitioned Window

```sql
-- Ranking within each group
SELECT
    category_id,
    name,
    price,
    ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) AS ranking
FROM products;
```

---

## 8. Practical Query Patterns

### Deduplication Queries

```sql
-- Method 1: DISTINCT
SELECT DISTINCT category_id FROM products;

-- Method 2: GROUP BY
SELECT category_id FROM products GROUP BY category_id;

-- Method 3: Window function (get the latest record per group)
SELECT * FROM (
    SELECT *,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) t WHERE rn = 1;
```

### Percentage Calculation

```sql
SELECT
    status,
    COUNT(*) AS cnt,
    COUNT(*) * 100.0 / SUM(COUNT(*)) OVER() AS percentage
FROM orders
GROUP BY status;
```

### Consecutive Login Users

```sql
-- Assuming login_logs(user_id, login_date)
SELECT DISTINCT user_id
FROM (
    SELECT
        user_id,
        login_date,
        DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (
            PARTITION BY user_id ORDER BY login_date
        ) DAY) AS group_date
    FROM login_logs
) t
GROUP BY user_id, group_date
HAVING COUNT(*) >= 3;
```

---

## Summary

This article delved into WHERE filtering, sorting, grouping, pagination, and conditional expressions. Mastering these advanced query techniques will cover most daily SQL needs.

---

