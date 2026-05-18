---
title: "MySQL 从入门到精通 · 第13篇：SQL 注入防御与安全实践"
date: 2025-01-17
weight: 13
draft: false
tags: ["mysql"]
---

## 什么是 SQL 注入？

SQL 注入是攻击者通过在输入中嵌入恶意 SQL 代码，欺骗数据库执行非预期的查询。

### 攻击示例

```python
# 存在注入风险的代码
username = request.GET['username']  # 用户输入："'; DROP TABLE users; -- "
sql = f"SELECT * FROM users WHERE username = '{username}'"

# 实际执行的 SQL：
# SELECT * FROM users WHERE username = ''; DROP TABLE users; -- '
```

### 注入后果

| 后果 | 说明 |
|------|------|
| 数据泄露 | 读取敏感数据 |
| 数据篡改 | 修改或删除数据 |
| 权限提升 | 绕过认证 |
| 远程命令 | 通过 `INTO OUTFILE` 写 Webshell |

---

## 一、防御原则

```
防御优先级：
1. 预编译参数化查询（最有效）
2. 输入验证和过滤
3. 最小数据库权限
4. 数据库安全配置
```

---

## 二、预编译（Prepared Statement）

预编译将 SQL 语句结构和参数分离，参数被当作数据而非 SQL 执行，从根本上杜绝注入。

### Java (JDBC)

```java
// ❌ 不安全：字符串拼接
String sql = "SELECT * FROM users WHERE name = '" + name + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);

// ✅ 安全：预编译
String sql = "SELECT * FROM users WHERE name = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, name);
ResultSet rs = pstmt.executeQuery();
```

### Python (PyMySQL)

```python
# ❌ 不安全
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ 安全：参数化查询
cursor.execute("SELECT * FROM users WHERE name = %s", (name,))
```

### Node.js (mysql2)

```javascript
// ✅ 安全：占位符
const sql = 'SELECT * FROM users WHERE name = ?';
connection.query(sql, [name], (err, results) => { ... });
```

### PHP (PDO)

```php
// ✅ 安全：绑定参数
$stmt = $pdo->prepare("SELECT * FROM users WHERE name = :name");
$stmt->execute([':name' => $name]);
```

### Go (database/sql)

```go
// ✅ 安全
rows, err := db.Query("SELECT * FROM users WHERE name = ?", name)
```

---

## 三、ORM 防护

ORM 框架通常默认使用参数化查询：

```python
# Django ORM（安全）
User.objects.filter(name=name)

# SQLAlchemy（安全）
session.query(User).filter(User.name == name).all()
```

```javascript
// Sequelize（安全）
User.findAll({ where: { name: name } });

// Prisma（安全）
prisma.user.findMany({ where: { name: name } });
```

```java
// MyBatis（安全）
@Select("SELECT * FROM users WHERE name = #{name}")
User findByName(String name);
```

**但 ORM 也有坑**：

```python
# ❌ 危险！原生 SQL 查询
User.objects.raw(f"SELECT * FROM users WHERE name = '{name}'")
```

```javascript
// ❌ 危险！Sequelize 原生查询
sequelize.query(`SELECT * FROM users WHERE name = '${name}'`);
```

使用 ORM 时，原生查询一定要用参数化方式。

---

## 四、其他注入点防御

### LIKE 查询

```java
// LIKE 中的 % 需要特殊处理
name = name.replace("!", "!!")
         .replace("%", "!%")
         .replace("_", "!_");
String sql = "SELECT * FROM users WHERE name LIKE ? ESCAPE '!'";
// 注意：预编译依然保护了 SQL 结构，ESCAPE 处理的是搜索逻辑问题
```

### IN 查询

```java
// 动态 IN 列表需要用占位符展开
List<String> ids = Arrays.asList("1", "2", "3");
StringBuilder placeholders = new StringBuilder();
for (int i = 0; i < ids.size(); i++) {
    if (i > 0) placeholders.append(",");
    placeholders.append("?");
}
String sql = "SELECT * FROM users WHERE id IN (" + placeholders + ")";
PreparedStatement pstmt = conn.prepareStatement(sql);
for (int i = 0; i < ids.size(); i++) {
    pstmt.setString(i + 1, ids.get(i));
}
```

### ORDER BY 和 表名

ORDER BY 和表名不能参数化，需要白名单验证：

```java
// ❌ 不安全
String sql = "SELECT * FROM users ORDER BY " + sortColumn;

// ✅ 安全：白名单验证
List<String> validColumns = Arrays.asList("id", "name", "created_at");
if (!validColumns.contains(sortColumn)) {
    sortColumn = "id";  // 默认值
}
String sql = "SELECT * FROM users ORDER BY " + sortColumn;
// 注意：这里的 sortColumn 经过了白名单校验，不是用户直接输入
```

---

## 五、数据库层面的安全配置

### 最小权限原则

```sql
-- ❌ 危险：给应用账号 DDL 权限
GRANT ALL PRIVILEGES ON shop.* TO 'app'@'%';

-- ✅ 安全：只给 DML 权限
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app'@'%';

-- ✅ 更安全：只给需要操作的表
GRANT SELECT, INSERT, UPDATE ON shop.orders TO 'app'@'%';
```

### FILE 权限

```sql
-- ❌ 绝对不要给应用账号 FILE 权限
-- FILE 允许 SELECT ... INTO OUTFILE 写文件到服务器

-- 检查哪些用户有 FILE 权限
SELECT User, Host FROM mysql.user
WHERE File_priv = 'Y';
```

### local_infile

```ini
# my.cnf
[mysqld]
local-infile = 0
```

禁用 `LOAD DATA LOCAL INFILE`，防止攻击者读取服务器文件。

---

## 六、其他安全实践

### 1. 输入验证

```java
// 数字类型
int id = Integer.parseInt(request.getParameter("id"));  // 严格类型转换

// 字符串
String email = request.getParameter("email");
if (!email.matches("^[\\w.+-]+@[\\w-]+\\.[\\w.]+$")) {
    throw new IllegalArgumentException("Invalid email");
}
```

### 2. 输出编码

```python
# 在网页上显示用户输入时做 HTML 转义
import html
safe_output = html.escape(user_input)
```

### 3. 隐藏数据库错误

```python
# ❌ 危险：将数据库错误直接暴露给用户
except MySQLdb.Error as e:
    return f"Database error: {e}"

# ✅ 安全：记录日志，返回通用错误
except MySQLdb.Error as e:
    logger.error(f"Database error: {e}")
    return "An error occurred. Please try again later."
```

---

## 七、SQL 注入检测

### 自动化检测

```bash
# 使用 sqlmap 进行渗透测试（仅在授权范围内）
sqlmap -u "http://example.com/page?id=1" --batch

# 静态代码扫描（以 Semgrep 为例）
semgrep --config=p/security-audit --language=python .
```

### 代码审查

检查以下模式：

```python
# 🔴 危险模式（需要在代码审查中重点关注）：
cursor.execute(f"SELECT ... WHERE name = '{name}'")   # f-string 拼接
cursor.execute("SELECT ... WHERE name = '" + name + "'")  # 字符串拼接
cursor.execute("SELECT ... WHERE name = %s" % name)       # % 格式化
```

---

## 八、理解 SQL 注入的工作原理

```python
# 假设验证登录的 SQL：
sql = f"SELECT * FROM users WHERE name = '{username}' AND password = '{password}'"

# 攻击者输入：
username = "admin"
password = "' OR '1'='1"
# 实际执行的 SQL：
# SELECT * FROM users WHERE name = 'admin' AND password = '' OR '1'='1'
# '1'='1' 永远为真，绕过认证！
```

另一个经典的注入：

```python
username = "admin' -- "
# SELECT * FROM users WHERE name = 'admin' -- ' AND password = 'xxx'
# -- 是 SQL 注释，后面的条件被注释掉
```

---

## 九、安全的连接信息

```python
# ❌ 危险：硬编码数据库密码
DB_PASSWORD = "password123"

# ✅ 安全：使用环境变量
import os
DB_PASSWORD = os.getenv("DB_PASSWORD")

# 或者使用专门的密钥管理服务
# AWS Secrets Manager / HashiCorp Vault
```

### 配置文件安全

```bash
# .env 文件不要提交到版本控制
echo ".env" >> .gitignore

# 配置文件设置最小权限
chmod 600 config/database.yml
```

---

## 十、安全清单

```
SQL 注入防御检查清单：
□ 所有 SQL 使用参数化查询或预编译
□ ORM 使用时避免原生 SQL 拼接
□ LIKE/IN/ORDER BY 特殊场景正确处理
□ 应用账号只具备最小必要权限
□ 禁止 FILE 权限
□ 禁用 local_infile
□ 数据库错误信息不暴露给用户
□ 敏感配置使用环境变量或密钥管理
□ 定期代码审查扫描注入风险
□ 渗透测试验证注入防护
```

---

## 小结

SQL 注入是最经典的 Web 安全漏洞之一。防御的关键不在于过滤多严格，而在于**从根本上将数据和代码分离**——参数化查询就是最佳实践。配合最小权限和安全配置，能有效防御绝大多数的注入攻击。

---

