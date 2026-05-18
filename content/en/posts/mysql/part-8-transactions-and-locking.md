---
title: "MySQL from Beginner to Pro · Part 8: Transactions and Locking"
date: 2025-01-28
weight: 1308
draft: false
tags: ["mysql"]
---

## 1. What is a Transaction?

A transaction is a group of SQL operations that either all succeed or all fail and roll back.

```sql
-- Bank transfer example
START TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;

COMMIT;  -- Commit
-- or
ROLLBACK;  -- Rollback
```

If the server crashes after the first UPDATE succeeds, the transaction mechanism ensures the second UPDATE will not execute, maintaining data consistency.

---

## 2. ACID Properties

| Property | Meaning | Implementation |
|------|------|---------|
| **A**tomicity | Transaction is indivisible, all or nothing | undo log |
| **C**onsistency | Data satisfies all constraints before and after transaction | Application + DB constraints |
| **I**solation | Concurrent transactions don't interfere with each other | Locks + MVCC |
| **D**urability | Committed data is permanently saved | redo log |

---

## 3. Transaction Isolation Levels

SQL standard defines four isolation levels, from lowest to highest:

```sql
-- View current isolation level
SELECT @@transaction_isolation;
-- MySQL 8.0 default: REPEATABLE-READ

-- Set isolation level
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Description |
|---------|------|-----------|------|------|
| `READ UNCOMMITTED` | ✅ Possible | ✅ Possible | ✅ Possible | Can read uncommitted data |
| `READ COMMITTED` | ❌ | ✅ Possible | ✅ Possible | Only read committed (Oracle default) |
| `REPEATABLE READ` | ❌ | ❌ | ✅ Possible | Consistent reads within a transaction (MySQL default) |
| `SERIALIZABLE` | ❌ | ❌ | ❌ | Serializable, safest but slowest |

### Dirty Read

Transaction A reads **uncommitted** data from Transaction B. If B rolls back, A has read non-existent data.

### Non-repeatable Read

Two reads of the same row within a transaction return different results.

```
Transaction A: SELECT name FROM users WHERE id = 1 → '张三'
Transaction B: UPDATE users SET name = '李四' WHERE id = 1; COMMIT;
Transaction A: SELECT name FROM users WHERE id = 1 → '李四'
```

### Phantom Read

Two range queries within a transaction return different numbers of rows.

```
Transaction A: SELECT * FROM orders WHERE amount > 100  → 5 rows
Transaction B: INSERT INTO orders (amount) VALUES (200); COMMIT;
Transaction A: SELECT * FROM orders WHERE amount > 100  → 6 rows (1 more)
```

---

## 4. MVCC (Multi-Version Concurrency Control)

InnoDB implements the `REPEATABLE READ` level through **MVCC**, solving the non-repeatable read problem **without locking**.

### Core Principle

Each row of data may have multiple versions (multiple snapshots), and each transaction sees the data version from its own "snapshot" moment.

```sql
-- A data row actually contains two hidden columns:
-- DB_TRX_ID: Transaction ID that last modified the row
-- DB_ROLL_PTR: Pointer to undo log, used to find older versions
```

### Read View

A transaction generates a **Read View** at its start, recording the list of active transactions at that time:

```
Read View
├── creator_trx_id: Current transaction ID
├── m_ids: List of active transaction IDs
├── min_trx_id: Minimum active transaction ID
└── max_trx_id: Next transaction ID to be assigned
```

**Visibility Rules**:

```
DB_TRX_ID of a data row:
├── < min_trx_id       → Committed, visible
├── = creator_trx_id   → Modified by itself, visible
├── > max_trx_id       → Future transaction, not visible
├── In m_ids           → Uncommitted, not visible
└── Not in m_ids       → Committed, visible
```

### What MVCC Solves

- ✅ Dirty Read: Uncommitted modifications are invisible to other transactions
- ✅ Non-repeatable Read: Multiple queries within the same transaction use the same Read View
- ❌ Phantom Read: MVCC cannot fully solve it (but InnoDB partially solves it with gap locks)

---

## 5. Row Lock (Record Lock)

Locks index records, preventing other transactions from modifying or deleting.

```sql
-- Shared Lock (S Lock): Allows multiple transactions to read, prevents writes
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;

-- Exclusive Lock (X Lock): Prevents other transactions from reading and writing
SELECT * FROM users WHERE id = 1 FOR UPDATE;
```

### Three Row Lock Algorithms

```sql
-- InnoDB row locks are implemented based on indexes, locking index records
```

**Record Lock**: Single row record lock

```sql
-- Lock the row with id = 1
SELECT * FROM users WHERE id = 1 FOR UPDATE;
```

**Gap Lock**: Locks the gap between records, preventing inserts

```sql
-- Assuming users table has ids: 1, 3, 5
SELECT * FROM users WHERE id BETWEEN 2 AND 4 FOR UPDATE;
-- Locks gaps (1,3) and (3,5), cannot insert id=2 or id=4
```

**Next-Key Lock**: Record Lock + Gap Lock combination

```sql
-- Default locking mechanism at InnoDB RR level
-- Locks the record itself + the gap before it
```

---

## 6. Table Locks

### Table-Level Locks

```sql
-- Manually lock tables
LOCK TABLES users READ;     -- Read lock, other sessions can read but not write
LOCK TABLES users WRITE;    -- Write lock, other sessions cannot read or write

UNLOCK TABLES;
```

### Intention Locks

InnoDB manages these automatically with no manual intervention needed. When a transaction needs to add a row lock to a table, it automatically adds an intention lock at the table level to quickly determine if the table is already locked by another transaction.

```
Intention Shared Lock (IS): Preparing to add shared locks to certain rows
Intention Exclusive Lock (IX): Preparing to add exclusive locks to certain rows
```

---

## 7. Deadlock

Two transactions waiting for each other to release locks:

```sql
-- Transaction A
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Locks id=1
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Waits for B to release id=2

-- Transaction B
START TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 2;  -- Locks id=2
UPDATE accounts SET balance = balance + 100 WHERE id = 1;  -- Waits for A to release id=1
```

### InnoDB Deadlock Handling

```sql
-- View deadlock information
SHOW ENGINE INNODB STATUS\G

-- InnoDB automatically detects deadlocks and rolls back one of the transactions (the one with lower rollback cost)
```

### Avoiding Deadlocks

```sql
-- 1. Access resources in the same order
-- ❌ May deadlock
UPDATE accounts SET ... WHERE id = 1;
UPDATE accounts SET ... WHERE id = 2;
-- Another transaction:
UPDATE accounts SET ... WHERE id = 2;
UPDATE accounts SET ... WHERE id = 1;

-- ✅ Fixed order
UPDATE accounts SET ... WHERE id = 1;
UPDATE accounts SET ... WHERE id = 2;
-- Another transaction also follows id ascending order

-- 2. Shorten transaction duration
-- 3. Use READ COMMITTED to reduce gap locks
```

---

## 8. Transaction Best Practices

### Proper Error Handling

```sql
-- Stored procedure
CREATE PROCEDURE transfer(IN from_id INT, IN to_id INT, IN amount DECIMAL(10,2))
BEGIN
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        RESIGNAL;
    END;

    START TRANSACTION;

    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;

    COMMIT;
END;
```

### Setting Timeouts

```sql
-- Lock wait timeout (default 50 seconds)
SET innodb_lock_wait_timeout = 10;  -- 10 seconds

-- Transaction timeout
SET SESSION max_execution_time = 5000;  -- 5 seconds
```

### Viewing Lock Information

```sql
-- View current lock waits
SELECT * FROM performance_schema.data_lock_waits\G

-- View transactions
SELECT * FROM information_schema.INNODB_TRX\G
```

---

## 9. Isolation Level Selection Recommendations

| Scenario | Recommended Isolation Level | Reason |
|------|-------------|------|
| Default | REPEATABLE READ | MySQL default, good MVCC support |
| High concurrency | READ COMMITTED | Reduces gap locks, improves concurrency |
| Financial systems | REPEATABLE READ or SERIALIZABLE | High data consistency requirements |
| Report analytics | READ UNCOMMITTED | Allows dirty reads, reduces lock waits |

```sql
-- Recommended for high concurrency: READ COMMITTED + binlog_row_image=MINIMAL
SET GLOBAL transaction_isolation = 'READ-COMMITTED';
```

---

## Summary

Transactions and locking are the core mechanisms for ensuring data consistency. This article covered ACID, isolation levels, MVCC principles, row and table locks, deadlock handling, and other key concepts. Understanding these principles is crucial for writing correct, high-concurrency programs.

---

