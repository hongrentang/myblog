---
title: "MySQL 从入门到精通 · 第4篇：CRUD 操作详解"
date: 2025-01-24
weight: 4
draft: false
tags: ["mysql"]
---

CRUD 是数据库最基本的操作：**Create（创建）**、**Read（读取）**、**Update（更新）**、**Delete（删除）**。

---

## 一、INSERT —— 插入数据

### 基础语法

```sql
-- 插入单条（指定字段）
INSERT INTO users (name, email, age)
VALUES ('张三', 'zhangsan@example.com', 25);

-- 插入单条（所有字段，不推荐）
INSERT INTO users VALUES (1, '张三', 'zhangsan@example.com', 25, NOW(), NOW());

-- 插入多条
INSERT INTO users (name, email, age) VALUES
    ('李四', 'lisi@example.com', 30),
    ('王五', 'wangwu@example.com', 28),
    ('赵六', 'zhaoliu@example.com', 22);
```

### 高级用法

```sql
-- INSERT ... SELECT（从另一张表查询并插入）
INSERT INTO vip_users (name, email)
SELECT name, email FROM users WHERE age > 25;

-- INSERT IGNORE（重复则忽略，不报错）
INSERT IGNORE INTO users (id, name, email) VALUES (1, '张三', 'zhangsan@example.com');

-- REPLACE（有则替换，无则插入）
REPLACE INTO users (id, name, email) VALUES (1, '张三2', 'zhangsan2@example.com');

-- ON DUPLICATE KEY UPDATE（重复时更新）
INSERT INTO users (id, name, email) VALUES (1, '张三', 'zhangsan@example.com')
ON DUPLICATE KEY UPDATE name = VALUES(name), email = VALUES(email);
```

### INSERT 性能优化

```sql
-- 大批量插入时，可以分段提交
INSERT INTO logs (message) VALUES ('msg1'), ('msg2'), ...;  -- 一次 500~1000 条

-- 临时关闭唯一性检查（导入后重新开启）
SET UNIQUE_CHECKS = 0;
-- 执行大量 INSERT ...
SET UNIQUE_CHECKS = 1;
```

---

## 二、SELECT —— 查询数据

### 基础查询

```sql
-- 查询所有字段（尽量避免在生产使用）
SELECT * FROM users;

-- 查询指定字段
SELECT id, name, email FROM users;

-- 别名
SELECT name AS 姓名, email AS 邮箱 FROM users;

-- 去重
SELECT DISTINCT status FROM orders;

-- 常量列
SELECT name, 'VIP' AS user_type FROM users;
```

### 条件查询

```sql
-- 比较运算符
SELECT * FROM users WHERE age >= 18;
SELECT * FROM orders WHERE status != 0;

-- 范围
SELECT * FROM users WHERE age BETWEEN 18 AND 35;
SELECT * FROM orders WHERE total_amount > 100;

-- IN 列表
SELECT * FROM users WHERE id IN (1, 3, 5, 7);

-- 模糊查询
SELECT * FROM users WHERE name LIKE '张%';    -- 以"张"开头
SELECT * FROM users WHERE email LIKE '%@gmail.com';  -- gmail 邮箱

-- NULL 判断
SELECT * FROM users WHERE phone IS NULL;
SELECT * FROM users WHERE phone IS NOT NULL;
```

---

## 三、UPDATE —— 更新数据

### 基础语法

```sql
-- 更新单字段
UPDATE users SET age = 26 WHERE id = 1;

-- 更新多字段
UPDATE users SET age = 26, phone = '13800138000' WHERE id = 1;

-- 不加 WHERE 会更新全表！
UPDATE users SET status = 1;  -- ⚠️ 全表更新
```

### 条件更新

```sql
-- 更新符合条件的数据
UPDATE orders SET status = 1, paid_at = NOW()
WHERE status = 0 AND created_at < DATE_SUB(NOW(), INTERVAL 30 MINUTE);

-- 使用 CASE WHEN 批量更新
UPDATE users SET age = CASE id
    WHEN 1 THEN 20
    WHEN 2 THEN 25
    WHEN 3 THEN 30
END
WHERE id IN (1, 2, 3);
```

### UPDATE 注意事项

```sql
-- 1. 更新时利用原字段值
UPDATE products SET stock = stock - 1 WHERE id = 1;
-- 比先 SELECT 再 UPDATE 更安全、更高效（原子操作）

-- 2. 带 LIMIT 限制影响行数
UPDATE users SET status = 1 WHERE status = 0 LIMIT 100;

-- 3. UPDATE 结合 JOIN
UPDATE orders o
JOIN users u ON o.user_id = u.id
SET o.status = 1
WHERE u.email = 'zhangsan@example.com';
```

---

## 四、DELETE —— 删除数据

### 基本语法

```sql
-- 删除指定行
DELETE FROM users WHERE id = 1;

-- 删除所有行（但自增值不回重置）
DELETE FROM temp_logs;

-- 带 LIMIT 安全删除
DELETE FROM logs WHERE created_at < '2024-01-01' LIMIT 1000;
```

### 安全删除模式

```sql
-- 先确认要删的数据
SELECT * FROM users WHERE email = 'bad@example.com';

-- 确认无误后删除
DELETE FROM users WHERE email = 'bad@example.com';

-- 或者使用软删除
ALTER TABLE users ADD COLUMN deleted_at DATETIME DEFAULT NULL;
-- "删除"时：
UPDATE users SET deleted_at = NOW() WHERE id = 1;
-- 查询时过滤掉已删除：
SELECT * FROM users WHERE deleted_at IS NULL;
```

**软删除（逻辑删除）** 是推荐做法，数据不丢失，可恢复。

---

## 五、分页查询

```sql
-- LIMIT 偏移量, 数量
SELECT * FROM users ORDER BY id LIMIT 0, 20;   -- 第1页
SELECT * FROM users ORDER BY id LIMIT 20, 20;  -- 第2页
SELECT * FROM users ORDER BY id LIMIT 40, 20;  -- 第3页

-- LIMIT 数量 OFFSET 偏移量（等价写法）
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 40;
```

### 深分页优化

```sql
-- ❌ 传统深分页（越往后越慢）
SELECT * FROM users ORDER BY id LIMIT 100000, 20;

-- ✅ 游标分页（推荐）
SELECT * FROM users WHERE id > 100000 ORDER BY id LIMIT 20;

-- ✅ 子查询优化
SELECT * FROM users
WHERE id >= (SELECT id FROM users ORDER BY id LIMIT 100000, 1)
ORDER BY id LIMIT 20;
```

---

## 六、聚合查询

```sql
-- 常用聚合函数
SELECT
    COUNT(*) AS total_users,
    AVG(age) AS avg_age,
    MAX(age) AS max_age,
    MIN(age) AS min_age,
    SUM(balance) AS total_balance
FROM users;

-- 分组统计
SELECT status, COUNT(*) AS cnt, SUM(total_amount) AS total
FROM orders
GROUP BY status;

-- HAVING 对聚合结果过滤
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING order_count > 5;

-- GROUP_CONCAT 合并
SELECT user_id, GROUP_CONCAT(order_no SEPARATOR ',') AS orders
FROM orders
GROUP BY user_id;
```

---

## 七、排序

```sql
-- 升序
SELECT * FROM products ORDER BY price ASC;

-- 降序
SELECT * FROM products ORDER BY price DESC;

-- 多字段排序
SELECT * FROM products ORDER BY category_id, price DESC;

-- 按表达式排序
SELECT * FROM products ORDER BY price * discount ASC;

-- 按字段值排序（自定义排序）
SELECT * FROM users ORDER BY FIELD(status, 'active', 'pending', 'disabled');
```

---

## 八、完整查询顺序

```sql
SELECT DISTINCT 字段              -- 5. 选择输出字段
FROM 表                           -- 1. 确定数据源
JOIN 其他表 ON 条件               -- 2. 多表连接
WHERE 条件                        -- 3. 行过滤
GROUP BY 字段                     -- 4. 分组
HAVING 聚合条件                   -- 4.5 分组后过滤
ORDER BY 字段                     -- 6. 排序
LIMIT 数量 OFFSET 偏移;           -- 7. 分页
```

理解这个执行顺序对写正确 SQL 很有帮助。

---

## 实战练习

以订单系统为例，综合运用 CRUD：

```sql
-- 1. 注册用户
INSERT INTO users (name, email, phone) VALUES ('张三', 'zs@test.com', '13800001111');

-- 2. 用户下单
INSERT INTO orders (order_no, user_id, total_amount, status)
VALUES ('20250115001', '1', 299.00, 0);

-- 3. 查询用户订单
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.email = 'zs@test.com'
ORDER BY o.created_at DESC;

-- 4. 支付订单
UPDATE orders SET status = 1, paid_at = NOW()
WHERE order_no = '20250115001';

-- 5. 取消过期未支付订单
UPDATE orders SET status = 4
WHERE status = 0 AND created_at < NOW() - INTERVAL 1 DAY;

-- 6. 统计用户消费排行
SELECT u.name, COUNT(o.id) AS order_count, SUM(o.total_amount) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 1
GROUP BY u.id, u.name
ORDER BY total_spent DESC
LIMIT 10;
```

---

## 小结

这一篇覆盖了 CRUD 的完整用法，包括插入、查询、更新、删除、分页和聚合。熟练掌握这些操作是数据库开发的基本功。

---

