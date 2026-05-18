---
title: "MySQL from Beginner to Pro · Part 7: Index Principles and Optimization"
date: 2025-01-27
weight: 1307
draft: false
tags: ["mysql"]
---

## Why Do We Need Indexes?

Queries without indexes are called **full table scans** -- MySQL reads the entire table from beginning to end. An index is like a book's table of contents, allowing you to quickly locate content by looking up the directory.

```
Full table scan: Flip through the entire book to find one page
Index lookup: Check the directory → Go directly to that page
```

---

## 1. Index Data Structure: B+ Tree

MySQL InnoDB uses **B+ Tree** as its index structure.

```
                  [Root Node]
              /      |      \
         [Branch] [Branch] [Branch]
         /   \    /   \    /   \
      [Leaf] [Leaf][Leaf][Leaf][Leaf]
       Data   Data  Data  Data  Data
```

### B+ Tree Features

| Feature | Description |
|------|------|
| Multi-way balanced tree | Each node can store multiple keys, tree height is very low |
| Leaf nodes store data | All data resides in leaf nodes |
| Leaf nodes have linked list | Facilitates range queries (>, <, BETWEEN) |
| Non-leaf nodes store only keys | No data, one page can hold more keys |

### Query Cost

InnoDB's B+ Tree height is typically **2~4 levels** (3 levels is sufficient for millions of records). One index lookup = reading 2~4 disk pages, which is an order of magnitude improvement over full table scans.

---

## 2. Clustered Index vs Secondary Index

### Clustered Index

In InnoDB, the **primary key is the clustered index**. Leaf nodes directly store the entire row of data.

```
Clustered Index (Primary Key id)
[1: 张三...] → [2: 李四...] → [3: 王五...] → ...
```

- Each table has exactly one clustered index
- If no primary key is defined, InnoDB selects the first UNIQUE column as the clustered index
- If neither exists, InnoDB automatically generates a hidden ROW_ID

### Secondary Index

Non-primary key indexes are secondary indexes. Leaf nodes store the **primary key value**.

```
Secondary Index (name)
[张三: 1] → [李四: 2] → [王五: 3] → ...
```

### Back to Table Query

```sql
CREATE INDEX idx_name ON users(name);

-- Query process (back to table):
SELECT * FROM users WHERE name = '张三';
-- 1. Find primary key id = 1 through idx_name
-- 2. Go back to clustered index with id = 1 to find the full row
```

### Covering Index

If all queried fields are in the secondary index, no back-to-table is needed:

```sql
-- Only need name and id, idx_name already contains them → no back-to-table needed
SELECT id, name FROM users WHERE name = '张三';
```

---

## 3. Index Types

### Normal Index

```sql
CREATE INDEX idx_email ON users(email);
```

### Unique Index

```sql
CREATE UNIQUE INDEX idx_email ON users(email);
-- Adds a uniqueness constraint on top of a normal index, similar query performance
```

### Composite Index

```sql
CREATE INDEX idx_name_age ON users(name, age);
```

**Leftmost Prefix Principle**:

```sql
idx_name_age(name, age)

-- ✅ Uses index
WHERE name = '张三'
WHERE name = '张三' AND age = 25
WHERE name LIKE '张%'

-- ❌ Does not use index (skipped name)
WHERE age = 25
WHERE age > 20 AND name = '张三'  -- Only uses the name part
```

### Prefix Index

```sql
-- Index only the first N characters of a string to save space
CREATE INDEX idx_email_prefix ON users(email(10));
```

### Full-Text Index

```sql
CREATE FULLTEXT INDEX idx_content ON articles(content);

-- Query using MATCH ... AGAINST
SELECT * FROM articles
WHERE MATCH(content) AGAINST('数据库优化' IN BOOLEAN MODE);
```

---

## 4. Index Optimization in Practice

### 1. Choosing Index Columns

```sql
-- ✅ Suitable for indexing
WHERE status = 1           -- High selectivity
WHERE user_id = 100        -- Frequently appears in WHERE
ORDER BY created_at        -- Frequently sorted
JOIN ... ON a.user_id = b.id  -- JOIN related columns

-- ❌ Not suitable for indexing
gender                     -- Only '男'/'女', too low selectivity
content                    -- Too large, use prefix index or full-text index
```

### 2. EXPLAIN for Index Analysis

```sql
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
```

| Column | Description | Good | Bad |
|----|------|----|----|
| `type` | Access type | const/ref/range | ALL |
| `possible_keys` | Possible indexes | Has value | NULL |
| `key` | Actually used index | Has value | NULL |
| `rows` | Scanned rows | Small | Large (millions) |
| `Extra` | Extra info | Using index | Using filesort |

**type from best to worst**:

```
system > const > eq_ref > ref > range > index > ALL
```

- `const`: Primary key or unique index equality lookup (fastest)
- `ref`: Non-unique index equality lookup
- `range`: Range scan
- `index`: Full index tree scan
- `ALL`: Full table scan (worst)

### 3. Common Index Issues

```sql
-- ❌ Function on index column (index失效)
WHERE DATE(created_at) = '2025-01-15'
-- ✅ Use range query instead
WHERE created_at >= '2025-01-15' AND created_at < '2025-01-16'

-- ❌ Implicit type conversion (index失效)
WHERE phone = 13800138000  -- phone is VARCHAR
-- ✅ Use string explicitly
WHERE phone = '13800138000'

-- ❌ LIKE with leading % (index失效)
WHERE name LIKE '%张'
-- ✅ Without leading %
WHERE name LIKE '张%'

-- ❌ OR with only one side indexed (index失效)
WHERE email = 'test@test.com' OR name = '张三'
-- ✅ Use UNION instead
SELECT * FROM users WHERE email = 'test@test.com'
UNION
SELECT * FROM users WHERE name = '张三'

-- ❌ != or NOT IN (typically doesn't use index)
WHERE status != 1
```

### 4. Index Maintenance

```sql
-- View index usage
SELECT
    index_name,
    cardinality,
    (cardinality / (SELECT COUNT(*) FROM users)) * 100 AS selectivity
FROM information_schema.statistics
WHERE table_schema = 'shop' AND table_name = 'users';

-- Analyze table (update statistics)
ANALYZE TABLE users;

-- View index size
SELECT
    database_name,
    table_name,
    index_name,
    ROUND(stat_value * @@innodb_page_size / 1024 / 1024, 2) AS size_mb
FROM mysql.innodb_index_stats
WHERE stat_name = 'size' AND table_name = 'users';
```

---

## 5. Index Design Principles

```
Index Design Checklist:
□ Primary key: Auto-increment BIGINT (ensures insert order, reduces page splits)
□ Frequently used WHERE columns
□ JOIN related columns (same types on both sides)
□ ORDER BY columns (reduces filesort)
□ High selectivity columns first (more unique values in front)
□ Control index count (no more than 5~8 per table)
□ Large fields use prefix indexes
□ Don't index every column
```

### Index Quantity Trade-off

| More Indexes | Fewer Indexes |
|--------|--------|
| Fast queries | Fast writes |
| Slow inserts/updates | Fast inserts/updates |
| More disk usage | Less disk usage |

**Core Principle**: Indexes are built to speed up queries -- don't create indexes for infrequently used queries.

---

## 6. Composite Index Leftmost Prefix in Detail

```sql
-- Create a composite index: (a, b, c)
CREATE INDEX idx_a_b_c ON t(a, b, c);

-- Queries that use the index
WHERE a = 1                          -- ✅ a
WHERE a = 1 AND b = 2                -- ✅ a, b
WHERE a = 1 AND b = 2 AND c = 3      -- ✅ a, b, c
WHERE a = 1 AND c = 3                -- ✅ a (but only a is used, c is not)
WHERE b = 2                          -- ❌ No index

-- Columns after range query don't use index
WHERE a = 1 AND b > 10 AND c = 3     -- ✅ a, b (c doesn't use index because b is a range)
```

---

## 7. MRR and ICP Optimization

### Index Condition Pushdown (ICP)

```sql
-- Enabled by default in MySQL 5.6+, pushes WHERE conditions down to the index layer for filtering
-- Without ICP: Back to table, then filter
-- With ICP: Filter at index layer first, reducing back-to-table calls
SET optimizer_switch = 'index_condition_pushdown=on';
```

### Multi-Range Read (MRR)

```sql
-- Sort by primary key before back-to-table, turning random I/O into sequential I/O
SET optimizer_switch = 'mrr=on,mrr_cost_based=on';
```

---

## 8. Practical Index Optimization Cases

```sql
-- Original query (slow)
SELECT * FROM orders
WHERE user_id = 100 AND status = 1
ORDER BY created_at DESC
LIMIT 10;

-- Analysis
EXPLAIN shows type: ref (user_id) but Extra: Using filesort

-- Solution 1: Create composite index
CREATE INDEX idx_user_status_time ON orders(user_id, status, created_at DESC);
-- Index order matches WHERE + ORDER BY, all use the index

-- Solution 2: If user_id alone returns too many records, narrow scope first
CREATE INDEX idx_user_time ON orders(user_id, created_at DESC);
```

---

## Summary

Indexes are the core of database performance. This article covered B+ Tree structure, clustered indexes, execution plan analysis, and index design principles. Remember: **EXPLAIN is your most faithful companion**.

---

