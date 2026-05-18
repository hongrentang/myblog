---
title: "MySQL from Beginner to Pro · Part 6: Multi-Table Queries"
date: 2025-01-26
weight: 1306
draft: false
tags: ["mysql"]
---

## Why Do We Need Multi-Table Queries?

In real business scenarios, data is not stored in a single table -- that would cause massive redundancy and maintenance difficulties. Through **foreign key** relationships, data is split across multiple tables, and **JOIN** is used to combine them during queries.

```
users                     orders                    products
┌─────────────┐      ┌──────────────────┐      ┌──────────────┐
│ id          │←──┐  │ id               │      │ id           │
│ name        │   └──│ user_id          │      │ name         │
│ email       │      │ product_id       │──→   │ price        │
│ created_at  │      │ quantity         │      │ stock        │
└─────────────┘      │ total_amount     │      └──────────────┘
                     │ created_at       │
                     └──────────────────┘
```

---

## 1. JOIN Types Overview

```sql
-- Prepare sample data
CREATE TABLE departments (
    id INT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    dept_id INT
);

INSERT INTO departments VALUES (1, '技术部'), (2, '市场部'), (3, '财务部'), (4, '运维部');
INSERT INTO employees VALUES (1, '张三', 1), (2, '李四', 1), (3, '王五', 2), (4, '赵六', NULL);
```

### INNER JOIN

Returns only matching rows from both tables:

```sql
SELECT e.name, d.name AS dept
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;

-- Result: 张三 Tech, 李四 Tech, 王五 Marketing
-- 赵六 has no department, not returned. Ops has no employees, not returned.
```

### LEFT JOIN

Returns all rows from the left table, NULL for non-matching right table:

```sql
SELECT e.name, d.name AS dept
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;

-- Result: 张三 Tech, 李四 Tech, 王五 Marketing, 赵六 NULL
```

### RIGHT JOIN

Returns all rows from the right table, NULL for non-matching left table:

```sql
SELECT e.name, d.name AS dept
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.id;

-- Result: 张三 Tech, 李四 Tech, 王五 Marketing, NULL Ops
```

### Memory Aid

```
INNER JOIN = Intersection of two circles
LEFT JOIN  = Left circle + intersection
RIGHT JOIN = Right circle + intersection
FULL JOIN  = Union of two circles (MySQL doesn't support, use UNION to simulate)
```

---

## 2. Multi-Table JOIN in Practice

### Three-Table Join

```sql
-- Query order information (including user and product)
SELECT
    o.id AS order_id,
    u.name AS user_name,
    p.name AS product_name,
    o.quantity,
    o.total_amount,
    o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN products p ON o.product_id = p.id;
```

### Self-Join

Joining a table with itself:

```sql
-- Employees and managers (assuming a manager_id field in the table)
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Query consecutive dates
SELECT a.date AS start_date, b.date AS end_date
FROM sales a
JOIN sales b ON DATEDIFF(b.date, a.date) = 1;
```

### Conditional JOIN

```sql
-- Only join orders from VIP users
SELECT u.name, o.total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 1
WHERE u.level = 'vip';
```

Put conditions in `ON` (doesn't change row count) rather than `WHERE` (which filters rows).

---

## 3. Subqueries

A subquery is a query nested inside another SQL statement.

### Subquery in WHERE

```sql
-- Find products with price above average
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);

-- Find users who have orders (IN)
SELECT * FROM users
WHERE id IN (SELECT DISTINCT user_id FROM orders);

-- Find users without orders (NOT IN, watch for NULL issues)
SELECT * FROM users
WHERE id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

### Subquery in FROM (Derived Table)

```sql
-- Find the highest priced product in each category
SELECT *
FROM products p
JOIN (
    SELECT category_id, MAX(price) AS max_price
    FROM products
    GROUP BY category_id
) t ON p.category_id = t.category_id AND p.price = t.max_price;
```

### Subquery in SELECT (Scalar Subquery)

```sql
SELECT
    u.name,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count,
    (SELECT SUM(total_amount) FROM orders o WHERE o.user_id = u.id AND o.status = 1) AS total_spent
FROM users u
WHERE u.status = 1;
```

---

## 4. EXISTS vs IN

```sql
-- IN
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders);

-- EXISTS
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

| Scenario | Recommendation |
|------|------|
| Small subquery result set | IN |
| Small outer table, large subquery | EXISTS |
| Need to correlate with outer table | EXISTS |
| NULL checks | EXISTS (IN has issues with NULL) |

---

## 5. UNION and UNION ALL

Combine results from multiple queries:

```sql
-- UNION (deduplicated)
SELECT name, email FROM vip_users
UNION
SELECT name, email FROM regular_users;

-- UNION ALL (not deduplicated, faster)
SELECT name, email FROM users WHERE city = '北京'
UNION ALL
SELECT name, email FROM users WHERE city = '上海';
```

**Rules:**
- Each SELECT must have the same number of columns
- Corresponding columns must have compatible data types
- `UNION` deduplicates (adds an extra sort step), `UNION ALL` does not
- Use `UNION ALL` instead of `UNION` when possible

---

## 6. Cartesian Product

```sql
-- ❌ Forgetting join conditions → Cartesian product (disaster!)
SELECT * FROM employees, departments;
-- 5 employees × 4 departments = 20 rows

-- ✅ Correct implicit join
SELECT * FROM employees e, departments d
WHERE e.dept_id = d.id;
```

---

## 7. Complex Query Examples

### Ranking

```sql
-- Top 3 products by sales in each category
SELECT category_id, product_id, total_sales
FROM (
    SELECT
        p.category_id,
        p.id AS product_id,
        SUM(o.quantity * o.price) AS total_sales,
        ROW_NUMBER() OVER (PARTITION BY p.category_id ORDER BY SUM(o.quantity * o.price) DESC) AS rn
    FROM products p
    JOIN order_items o ON p.id = o.product_id
    GROUP BY p.category_id, p.id
) t
WHERE rn <= 3;
```

### Recursive Query (CTE, 8.0+)

```sql
-- Query organizational hierarchy
WITH RECURSIVE org_tree AS (
    -- Root node
    SELECT id, name, parent_id, 1 AS level
    FROM org_units
    WHERE parent_id IS NULL

    UNION ALL

    -- Recursive child nodes
    SELECT u.id, u.name, u.parent_id, t.level + 1
    FROM org_units u
    JOIN org_tree t ON u.parent_id = t.id
)
SELECT * FROM org_tree ORDER BY level, id;
```

### Percentage Calculation

```sql
SELECT
    u.name,
    COUNT(o.id) AS order_count,
    COUNT(o.id) * 100.0 / SUM(COUNT(o.id)) OVER() AS percentage
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name
ORDER BY order_count DESC;
```

---

## 8. JOIN vs Subquery Comparison

| Aspect | JOIN | Subquery |
|--------|------|--------|
| Readability | Good, especially for multiple tables | Hard to read when deeply nested |
| Performance | Usually better (optimizer handles well) | Depends on specific scenario |
| Flexibility | Can return multiple columns | Scalar subqueries can only return one column |
| Recursion | ❌ | CTE supports recursion |

**Rule of thumb**: Prefer JOIN when it can solve the problem.

---

## Summary

Multi-table queries are a core capability of relational databases. This article covered INNER/LEFT/RIGHT JOIN, subqueries, EXISTS, UNION, and CTE recursive queries.

---

