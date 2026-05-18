---
title: "MySQL from Beginner to Pro · Part 16: Table Partitioning and Sharding"
date: 2025-01-20
weight: 1316
draft: false
tags: ["mysql"]
---

## When Do You Need Partitioning/Sharding?

```
Data Volume
├── Millions → Proper indexing is sufficient
├── Tens of millions → Table partitioning (single MySQL instance)
├── Hundreds of millions → Database sharding (multiple MySQL instances)
└── Billions+ → Distributed databases (TiDB, OceanBase)
```

---

## 1. Table Partitioning (Intra-Table)

Partitioning splits a large table into multiple physical sub-tables by rules, but remains transparent to the application (still appears as one table).

### Partition Types

#### 1. RANGE Partitioning

```sql
-- Partition by order year
CREATE TABLE orders (
    id BIGINT AUTO_INCREMENT,
    order_no VARCHAR(32),
    total_amount DECIMAL(10,2),
    created_at DATETIME,
    PRIMARY KEY (id, created_at)  -- Partition column must be part of the primary key
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

#### 2. LIST Partitioning (By Discrete Value List)

```sql
-- Partition by region
CREATE TABLE users (
    id INT,
    name VARCHAR(50),
    region VARCHAR(10),
    PRIMARY KEY (id, region)
) PARTITION BY LIST COLUMNS(region) (
    PARTITION p_north VALUES IN ('北京', '天津', '河北'),
    PARTITION p_east VALUES IN ('上海', '江苏', '浙江'),
    PARTITION p_south VALUES IN ('广东', '深圳', '福建')
);
```

#### 3. HASH Partitioning

```sql
-- Distribute by user ID evenly across 8 partitions
CREATE TABLE user_logs (
    id BIGINT,
    user_id INT,
    action VARCHAR(50),
    created_at DATETIME,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH(user_id) PARTITIONS 8;
```

#### 4. KEY Partitioning (Similar to HASH, uses MySQL's built-in hash function)

```sql
CREATE TABLE sessions (
    id INT PRIMARY KEY,
    session_id VARCHAR(64),
    data TEXT
) PARTITION BY KEY(session_id) PARTITIONS 10;
```

### Partition Query Optimization

```sql
-- Explicitly specify partition in queries (partition pruning)
SELECT * FROM orders PARTITION (p2024);

-- Use EXPLAIN to see if partition pruning is being used
EXPLAIN SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
-- If type is ALL but partitions only shows p2024, partition pruning is working
```

### Partition Management

```sql
-- Add partition
ALTER TABLE orders ADD PARTITION (
    PARTITION p2026 VALUES LESS THAN (2027)
);

-- Drop partition (fast partition-level deletion)
ALTER TABLE orders DROP PARTITION p2023;

-- Split partition
ALTER TABLE orders REORGANIZE PARTITION p_future INTO (
    PARTITION p2026 VALUES LESS THAN (2027),
    PARTITION p2027 VALUES LESS THAN (2028),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- View partition information
SELECT
    PARTITION_NAME,
    PARTITION_METHOD,
    PARTITION_EXPRESSION,
    TABLE_ROWS
FROM information_schema.PARTITIONS
WHERE TABLE_SCHEMA = 'shop' AND TABLE_NAME = 'orders';
```

### Partition Caveats

```sql
-- ❌ Partition column must be part of the primary key or unique index
CREATE TABLE bad (
    id INT PRIMARY KEY,
    created_at DATE
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p1 VALUES LESS THAN (2025)
);
-- Error: A PRIMARY KEY must include all columns in the table's partitioning function

-- ✅ Composite primary key including partition column
CREATE TABLE good (
    id INT,
    created_at DATE,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p1 VALUES LESS THAN (2025)
);
```

### When to Use Partitioning

| ✅ Suitable | ❌ Not Suitable |
|---------|----------|
| Large tables for time-based archiving | Tables with frequent join queries |
| Data grouped by region/category | Tables requiring unique primary key that cannot include partition column |
| Need fast time-range deletion of historical data | Tables with too many partitions (recommended ≤ 1024) |
| Tables where historical data is rarely modified | Scenarios with frequent cross-partition queries |

**Summary**: Partitioning is like adding physical "drawers" to a table, suitable for time-based archiving scenarios.

---

## 2. Database Sharding (Multi-Instance)

When a single table exceeds tens or hundreds of millions of records, and a single MySQL instance can no longer handle the load, **database sharding** is needed.

### Vertical Sharding

Splitting by business fields:

```sql
-- Original table: too many fields, not all needed in one query
users (
    id, name, email, phone, password_hash,
    avatar, bio, address, created_at, updated_at,
    login_ip, login_count, last_login_at  -- Infrequently queried
)

-- Split into:
users_base (id, name, email, phone, avatar, created_at)
users_auth (user_id, password_hash, login_ip, login_count, last_login_at)
```

### Horizontal Sharding

Distributing data across multiple tables with the same structure based on a key:

```sql
-- Split into 16 tables by user_id
user_0, user_1, user_2, ..., user_15

-- Routing rule:
table_index = user_id % 16

-- Query:
SELECT * FROM user_${user_id % 16} WHERE user_id = 100;
```

### Horizontal Database Sharding

Distributing tables across different MySQL instances:

```sql
-- Distribute across 4 databases by user_id
db0: user_0 ~ user_3
db1: user_4 ~ user_7
db2: user_8 ~ user_11
db3: user_12 ~ user_15

-- Routing:
db_index = user_id % 4
table_index = (user_id / 4) % 4
```

---

## 3. Sharding Middleware

### ShardingSphere (Recommended)

```yaml
# ShardingSphere-JDBC configuration example
spring:
  shardingsphere:
    datasource:
      names: ds0, ds1
      ds0:
        url: jdbc:mysql://192.168.1.100:3306/shop0
        username: app
        password: pass
      ds1:
        url: jdbc:mysql://192.168.1.101:3306/shop1
        username: app
        password: pass

    sharding:
      tables:
        users:
          actual-data-nodes: ds$->{0..1}.users_$->{0..1}
          table-strategy:
            standard:
              sharding-column: user_id
              sharding-algorithm-name: user-table-inline
          database-strategy:
            standard:
              sharding-column: user_id
              sharding-algorithm-name: user-db-inline
          key-generator:
            column: id
            type: SNOWFLAKE

    sharding-algorithms:
      user-db-inline:
        type: INLINE
        props:
          algorithm-expression: ds$->{user_id % 2}
      user-table-inline:
        type: INLINE
        props:
          algorithm-expression: users_$->{user_id % 2}
```

### MyCat

```xml
<!-- schema.xml -->
<schema name="shop" checkSQLschema="true" sqlMaxLimit="100">
    <table name="users" primaryKey="id" dataNode="dn0,dn1" rule="user-rule" />
</schema>

<dataNode name="dn0" dataHost="host1" database="shop0" />
<dataNode name="dn1" dataHost="host2" database="shop1" />

<dataHost name="host1" url="jdbc:mysql://192.168.1.100:3306" user="app" password="pass" />
<dataHost name="host2" url="jdbc:mysql://192.168.1.101:3306" user="app" password="pass" />
```

---

## 4. Sharding Strategies

### Mod Sharding

```sql
-- Simplest, but scaling requires data migration
db_index = user_id % 4
```

### Hash Sharding

```sql
-- Suitable for non-numeric primary keys
db_index = CRC32(order_no) % 8
```

### Range Sharding

```sql
-- By ID range, easy to scale
db0: user_id 1 ~ 10000000
db1: user_id 10000001 ~ 20000000
```

### Consistent Hashing

Reduces data migration during scaling, suitable for frequently changing clusters.

---

## 5. Challenges with Database Sharding

| Challenge | Description | Solution |
|------|------|---------|
| Cross-shard queries | Aggregate queries need to run across shards then merge | Avoid or minimize cross-shard operations |
| Distributed transactions | Consistency across database operations | Seata, TCC, eventual consistency |
| Global primary keys | Auto-increment IDs will duplicate | Snowflake, UUID (recommend ordered UUID v7) |
| Cross-shard sorting | Needs sorting per shard then merging | Middleware supports it, but minimize result sets |
| Join queries | Multi-table JOIN becomes difficult | Redundant fields, application-layer assembly |

### Global Primary Key Generation

```java
// Snowflake ID generation (Java example)
public class SnowflakeIdWorker {
    private long workerId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;

    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        if (timestamp < lastTimestamp) {
            throw new RuntimeException("Clock moved backwards");
        }
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & 4095;
            if (sequence == 0) {
                timestamp = waitNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0;
        }
        lastTimestamp = timestamp;
        return ((timestamp - 1288834974657L) << 22)
             | (workerId << 12)
             | sequence;
    }
}
```

---

## 6. Architecture Evolution Path

```sql
-- Phase 1: Single database, single table
Single MySQL instance

-- Phase 2: Read-write splitting
Master (write) + Slave (read)

-- Phase 3: Database sharding
Multiple MySQL instances + middleware

-- Phase 4: Distributed database
TiDB / OceanBase / PolarDB
```

**When do you need database sharding?**

```
1. Data volume > 50 million rows per table or > 100GB
2. MySQL write throughput becomes bottleneck (single instance can't handle write QPS)
3. Single instance connection count is insufficient
4. Splitting into multiple databases by business makes more sense
```

**Before reaching these thresholds, first optimize indexes, partitioning, and read-write splitting. Don't blindly adopt sharding.**

---

## 7. Partitioning vs Sharding Comparison

| Aspect | Partitioning | Database Sharding |
|------|------|---------|
| Complexity | Low (simple configuration) | High (requires middleware) |
| Application transparent | ✅ Fully transparent | ❌ Requires modification |
| Cross-shard queries | ✅ Automatic support | ❌ Limited |
| Transactions | ✅ Supported | ❌ Distributed transactions |
| Scalability | Limited by single instance | ✅ Virtually unlimited |
| Suitable data volume | Tens of millions ~ hundreds of millions | Hundreds of millions+ |

---

## Series Summary

| Part | Topic | Core Content |
|------|------|---------|
| 1 | MySQL Introduction and Installation | Installation, configuration, architecture |
| 2 | Database and Table Basics | CREATE, ALTER, DROP |
| 3 | Data Types and Constraints | Numeric/string/date types, constraint design |
| 4 | CRUD Operations | INSERT, SELECT, UPDATE, DELETE |
| 5 | Advanced Queries | WHERE, sorting, grouping, pagination, window functions |
| 6 | Multi-Table Queries | JOIN, subqueries, EXISTS, UNION |
| 7 | Index Principles and Optimization | B+ Tree, EXPLAIN, index strategies |
| 8 | Transactions and Locking | ACID, MVCC, isolation levels, deadlock |
| 9 | Stored Procedures and Functions | Stored procedures, functions, cursors |
| 10 | Triggers and Events | Triggers, scheduled events |
| 11 | Backup and Recovery | mysqldump, binlog, point-in-time recovery |
| 12 | User Privileges and Security | User creation, privilege system, SSL |
| 13 | SQL Injection Defense | Parameterized queries, prepared statements, security configuration |
| 14 | Performance Monitoring and Optimization | EXPLAIN ANALYZE, slow query log, tuning |
| 15 | Master-Slave Replication and Read-Write Splitting | binlog replication, GTID, ProxySQL |
| 16 | Table Partitioning and Sharding | Partitioning, sharding strategies, middleware |

Choose the right technical solution for your scenario, rather than picking the "most powerful" solution for every problem. For most applications, a single database with index optimization and read-write splitting is sufficient.

Hope this series has been helpful to you!

---

