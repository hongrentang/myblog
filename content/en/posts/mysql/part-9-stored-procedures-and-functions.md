---
title: "MySQL from Beginner to Pro · Part 9: Stored Procedures and Functions"
date: 2025-01-01
weight: 1309
draft: false
tags: ["mysql"]
---

## What are Stored Procedures?

A stored procedure is a **set of precompiled SQL statements stored in the database** that can be called like a function.

```sql
-- Create a stored procedure
DELIMITER //

CREATE PROCEDURE get_user_orders(IN user_id INT)
BEGIN
    SELECT * FROM orders WHERE user_id = user_id;
END //

DELIMITER ;

-- Call
CALL get_user_orders(1);
```

---

## Stored Procedure vs Function

| Comparison | Stored Procedure | Function |
|------|---------|------|
| Return value | None (can have OUT parameters) | Must have a return value |
| Call syntax | `CALL proc()` | `SELECT func()` |
| Transaction support | ✅ Can contain transactions | ❌ Cannot contain transactions |
| Use in SQL | ❌ | ✅ `WHERE func(x) > 0` |
| Output parameters | Supports IN/OUT/INOUT | Only RETURN |

---

## 1. Stored Procedures in Detail

### Parameter Modes

```sql
-- IN: Input parameter (default)
CREATE PROCEDURE get_user(IN p_id INT)
BEGIN
    SELECT * FROM users WHERE id = p_id;
END;

-- OUT: Output parameter
CREATE PROCEDURE count_users(OUT total INT)
BEGIN
    SELECT COUNT(*) INTO total FROM users;
END;

-- Calling OUT parameter
CALL count_users(@cnt);
SELECT @cnt;

-- INOUT: Input and output
CREATE PROCEDURE double_it(INOUT num INT)
BEGIN
    SET num = num * 2;
END;

SET @x = 10;
CALL double_it(@x);
SELECT @x;  -- 20
```

### Variables

```sql
CREATE PROCEDURE demo()
BEGIN
    -- Local variables
    DECLARE v_name VARCHAR(50);
    DECLARE v_count INT DEFAULT 0;

    -- Assignment
    SELECT name INTO v_name FROM users WHERE id = 1;
    SET v_count = 100;

    -- Use
    SELECT v_name, v_count;
END;
```

### Conditional Logic

```sql
CREATE PROCEDURE get_price_level(IN p_price DECIMAL(10,2), OUT p_level VARCHAR(20))
BEGIN
    IF p_price < 50 THEN
        SET p_level = '低价';
    ELSEIF p_price < 200 THEN
        SET p_level = '中等';
    ELSE
        SET p_level = '高价';
    END IF;
END;

-- CASE version
CREATE PROCEDURE get_status_text(IN p_status INT, OUT p_text VARCHAR(20))
BEGIN
    CASE p_status
        WHEN 0 THEN SET p_text = '待支付';
        WHEN 1 THEN SET p_text = '已支付';
        WHEN 2 THEN SET p_text = '已发货';
        ELSE SET p_text = '未知';
    END CASE;
END;
```

### Loops

```sql
CREATE PROCEDURE batch_insert()
BEGIN
    DECLARE i INT DEFAULT 0;

    WHILE i < 100 DO
        INSERT INTO test_log (message) VALUES (CONCAT('log_', i));
        SET i = i + 1;
    END WHILE;

    -- REPEAT version
    -- REPEAT
    --     SET i = i + 1;
    --     INSERT INTO test_log (message) VALUES (CONCAT('log_', i));
    -- UNTIL i >= 100 END REPEAT;

    -- LOOP version
    -- loop_label: LOOP
    --     SET i = i + 1;
    --     IF i > 100 THEN
    --         LEAVE loop_label;
    --     END IF;
    --     INSERT INTO test_log (message) VALUES (CONCAT('log_', i));
    -- END LOOP;
END;
```

### Cursors

```sql
CREATE PROCEDURE process_orders()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE v_order_id INT;
    DECLARE v_amount DECIMAL(10,2);

    -- Define cursor
    DECLARE cur CURSOR FOR SELECT id, total_amount FROM orders WHERE status = 0;
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cur;

    read_loop: LOOP
        FETCH cur INTO v_order_id, v_amount;
        IF done THEN
            LEAVE read_loop;
        END IF;

        -- Process expired orders
        UPDATE orders SET status = 4 WHERE id = v_order_id AND created_at < NOW() - INTERVAL 1 DAY;
    END LOOP;

    CLOSE cur;
END;
```

### Error Handling

```sql
CREATE PROCEDURE safe_transfer(IN from_id INT, IN to_id INT, IN amount DECIMAL(10,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SELECT '转账失败，已回滚' AS msg;
    END;

    START TRANSACTION;

    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    IF ROW_COUNT() = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '账户不存在';
    END IF;

    UPDATE accounts SET balance = balance + amount WHERE id = to_id;

    COMMIT;
    SELECT '转账成功' AS msg;
END;
```

---

## 2. Functions in Detail

### Scalar Functions

```sql
DELIMITER //

CREATE FUNCTION get_discount(price DECIMAL(10,2), user_level VARCHAR(10))
RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE discount_rate DECIMAL(3,2);

    CASE user_level
        WHEN 'vip' THEN SET discount_rate = 0.8;
        WHEN 'gold' THEN SET discount_rate = 0.9;
        ELSE SET discount_rate = 1.0;
    END CASE;

    RETURN ROUND(price * discount_rate, 2);
END //

DELIMITER ;

-- Usage
SELECT name, price, get_discount(price, 'vip') AS final_price FROM products;
```

### Function Characteristic Flags

```sql
CREATE FUNCTION ...
RETURNS type
-- Characteristic flags (important!)
DETERMINISTIC      -- Same input always produces same output
NOT DETERMINISTIC  -- May differ
READS SQL DATA     -- Read-only data
MODIFIES SQL DATA  -- Modifies data
NO SQL             -- No SQL
CONTAINS SQL       -- Contains SQL but doesn't read/write
```

### Table Functions (MySQL 8.0+)

```sql
-- JSON table function
SELECT * FROM JSON_TABLE('{"name":"张三","age":25}', '$'
    COLUMNS(
        name VARCHAR(10) PATH '$.name',
        age INT PATH '$.age'
    )
) AS jt;
```

---

## 3. Stored Procedure Management

```sql
-- View stored procedure status
SHOW PROCEDURE STATUS WHERE Db = 'shop';

-- View stored procedure definition
SHOW CREATE PROCEDURE get_user_orders;

-- View function definition
SHOW CREATE FUNCTION get_discount;

-- Drop
DROP PROCEDURE IF EXISTS get_user_orders;
DROP FUNCTION IF EXISTS get_discount;

-- Modify (DROP then CREATE, cannot directly ALTER)
```

---

## 4. Security and Considerations

### Pros and Cons

| Pros | Cons |
|------|------|
| Reduces network traffic (logic on DB side) | Hard to debug |
| Encapsulates complex logic | Version control unfriendly |
| Performance improvement (precompiled) | Migration difficulties |
| Unified business rules | Logic scattered across app and DB layers |

### Usage Recommendations

```sql
-- 1. Permissions: No special privileges needed by default
GRANT EXECUTE ON PROCEDURE shop.get_user_orders TO 'app_user'@'%';

-- 2. Don't overuse stored procedures
-- ✅ Suitable for: batch data processing, report generation
-- ❌ Not suitable for: simple CRUD (better in application layer)
```

---

## 5. Practical Examples

### Auto-generate Order Number

```sql
CREATE FUNCTION generate_order_no()
RETURNS VARCHAR(32)
DETERMINISTIC
BEGIN
    DECLARE date_part VARCHAR(8);
    DECLARE seq INT;
    DECLARE order_no VARCHAR(32);

    SET date_part = DATE_FORMAT(NOW(), '%Y%m%d');

    -- Get the maximum sequence number for the day (simple implementation)
    SELECT COALESCE(MAX(CAST(SUBSTRING(order_no, 9) AS UNSIGNED)), 0) + 1
    INTO seq
    FROM orders
    WHERE order_no LIKE CONCAT(date_part, '%');

    SET order_no = CONCAT(date_part, LPAD(seq, 6, '0'));

    RETURN order_no;
END;
```

### Monthly Sales Report

```sql
CREATE PROCEDURE generate_monthly_report(IN p_year INT, IN p_month INT)
BEGIN
    DECLARE start_date DATE;
    DECLARE end_date DATE;

    SET start_date = DATE(CONCAT(p_year, '-', p_month, '-01'));
    SET end_date = LAST_DAY(start_date);

    -- Create temporary table to store results
    CREATE TEMPORARY TABLE IF NOT EXISTS monthly_report (
        category_name VARCHAR(100),
        product_count INT,
        total_sales DECIMAL(12,2),
        total_profit DECIMAL(12,2)
    );

    -- Clear previous results
    TRUNCATE TABLE monthly_report;

    -- Populate data
    INSERT INTO monthly_report
    SELECT
        c.name,
        COUNT(DISTINCT oi.product_id),
        SUM(oi.quantity * oi.price),
        SUM(oi.quantity * (oi.price - p.cost))
    FROM categories c
    JOIN products p ON p.category_id = c.id
    JOIN order_items oi ON oi.product_id = p.id
    JOIN orders o ON o.id = oi.order_id
    WHERE o.status = 1
      AND o.paid_at >= start_date
      AND o.paid_at < DATE_ADD(end_date, INTERVAL 1 DAY)
    GROUP BY c.id, c.name
    ORDER BY total_sales DESC;

    -- Return results
    SELECT * FROM monthly_report;
END;

CALL generate_monthly_report(2025, 1);
```

---

## Summary

Stored procedures and functions encapsulate business logic in the database, suitable for batch data processing and complex calculation scenarios. But don't overuse them -- simple CRUD is better handled in the application layer.

---

