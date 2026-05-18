---
title: "MySQL 从入门到精通 · 第1篇：MySQL 简介与安装"
date: 2025-01-21
weight: 1
draft: false
tags: ["mysql"]
featured: true
cover:
  image: "/images/mysql-banner.svg"
  caption: "MySQL 从入门到精通"
---

## 什么是 MySQL？

MySQL 是一个开源的关系型数据库管理系统（RDBMS），由瑞典 MySQL AB 公司开发，后被 Sun 收购，最终归属于 Oracle。MySQL 是目前世界上最流行的开源数据库之一，在 Web 应用领域尤为常见。

**核心特点：**

- **关系型数据库**：数据以表（Table）的形式组织，表之间有关系（Relation）
- **SQL 标准**：支持标准 SQL 语法
- **ACID 事务**：支持原子性、一致性、隔离性、持久性
- **主从复制**：支持读写分离和高可用
- **插件式存储引擎**：InnoDB、MyISAM、Memory 等
- **跨平台**：Linux、Windows、macOS 都支持

---

## MySQL 与 MariaDB 的选择

MariaDB 是 MySQL 被 Oracle 收购后，由原作者 Monty 推出的分支。两者高度兼容：

| 对比 | MySQL | MariaDB |
|------|-------|---------|
| 当前版本 | 8.4+ / 9.x | 11.x |
| 维护方 | Oracle | MariaDB Foundation |
| 兼容性 | — | 完全兼容 MySQL API |
| 新特性 | HeatWave、JSON 增强 | 更多存储引擎、性能优化 |

大多数场景两者可互换。本系列以 **MySQL 8.4** 为例。

---

## 安装 MySQL

### Ubuntu/Debian

```bash
# 安装 MySQL 8.4
sudo apt update
sudo apt install mysql-server-8.4 -y

# 查看状态
sudo systemctl status mysql

# 安全配置（设置 root 密码、移除匿名用户等）
sudo mysql_secure_installation
```

### CentOS/RHEL

```bash
# 添加官方仓库
sudo rpm -ivh https://dev.mysql.com/get/mysql84-community-release-el9-1.noarch.rpm

# 安装
sudo yum install mysql-server -y

# 启动
sudo systemctl start mysqld
sudo systemctl enable mysqld

# 查看临时 root 密码
sudo grep 'temporary password' /var/log/mysqld.log
```

### Docker 安装

```bash
docker run --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=my-secret-pw \
  -p 3306:3306 \
  -d mysql:8.4
```

---

## 连接 MySQL

```bash
# 本地连接
mysql -u root -p

# 远程连接
mysql -h 192.168.1.100 -P 3306 -u root -p

# 连接后显示版本
mysql> SELECT VERSION();
+-----------+
| VERSION() |
+-----------+
| 8.4.2     |
+-----------+

# 查看当前所有数据库
mysql> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```

---

## MySQL 体系结构

```
客户端连接（连接池）
    ↓
SQL 接口（Parser → Optimizer → Executor）
    ↓
存储引擎层（InnoDB / MyISAM / Memory ...）
    ↓
文件系统（数据文件 + 日志文件）
```

### 关键组件

| 组件 | 作用 |
|------|------|
| **连接管理器** | 管理客户端连接，验证身份 |
| **查询解析器** | 解析 SQL 语句，生成解析树 |
| **查询优化器** | 选择最优执行计划 |
| **存储引擎** | 数据的存储和读取 |
| **缓冲池** | 缓存数据和索引，减少磁盘 I/O |
| **日志系统** | redo log（崩溃恢复）、binlog（复制与恢复）|

---

## 存储引擎对比

```sql
-- 查看支持的存储引擎
SHOW ENGINES;

-- 查看当前默认存储引擎
SELECT @@default_storage_engine;
+--------------------------+
| @@default_storage_engine |
+--------------------------+
| InnoDB                   |
+--------------------------+
```

| 特性 | InnoDB ✅ | MyISAM ❌ | Memory |
|------|-----------|-----------|--------|
| 事务支持 | ✅ | ❌ | ❌ |
| 行级锁 | ✅ | ❌（表锁）| ❌（表锁）|
| 外键 | ✅ | ❌ | ❌ |
| 崩溃恢复 | ✅ | ❌ | ❌ |
| MVCC | ✅ | ❌ | ❌ |
| 全文索引 | ✅（8.0+）| ✅ | ❌ |

> 在 MySQL 8.0+ 中，**InnoDB 是默认且推荐的存储引擎**。MyISAM 在新版本中已被标记为废弃。

---

## 第一个数据库

```sql
-- 创建数据库
CREATE DATABASE shop;
USE shop;

-- 创建表
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 插入数据
INSERT INTO users (name, email) VALUES ('张三', 'zhangsan@example.com');

-- 查询数据
SELECT * FROM users;
+----+--------+------------------------+---------------------+
| id | name   | email                  | created_at          |
+----+--------+------------------------+---------------------+
|  1 | 张三   | zhangsan@example.com   | 2025-01-15 10:00:00 |
+----+--------+------------------------+---------------------+
```

---

## 常用管理命令

```sql
-- 查看 MySQL 版本
SELECT VERSION();

-- 查看当前用户
SELECT USER();

-- 查看当前数据库
SELECT DATABASE();

-- 查看所有数据库
SHOW DATABASES;

-- 切换到数据库
USE database_name;

-- 查看数据库中的所有表
SHOW TABLES;

-- 查看表结构
DESC table_name;
DESCRIBE table_name;

-- 查看建表语句
SHOW CREATE TABLE table_name;

-- 查看 MySQL 配置
SHOW VARIABLES LIKE '%character%';
SHOW VARIABLES LIKE '%timeout%';
```

---

## 配置文件

MySQL 的配置文件路径：

```bash
# Linux
/etc/mysql/mysql.conf.d/mysqld.cnf
# 或
/etc/my.cnf

# 查看当前使用的配置文件
mysql --help | grep "Default options"
```

核心配置示例：

```ini
[mysqld]
# 基本
port = 3306
bind-address = 0.0.0.0
datadir = /var/lib/mysql

# 字符集
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# 存储引擎
default-storage-engine = InnoDB

# 连接
max_connections = 200
wait_timeout = 600

# 缓冲池（建议设为内存的 70%）
innodb_buffer_pool_size = 4G

# 日志
log_error = /var/log/mysql/error.log
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# binlog
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 7
```

修改配置后需要重启：

```bash
sudo systemctl restart mysql
```

---

## 小结

这一篇介绍了 MySQL 的安装、连接、体系结构和基本概念。

---

