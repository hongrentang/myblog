---
title: "MySQL from Beginner to Pro · Part 13: SQL Injection Defense"
date: 2025-01-17
weight: 1313
draft: false
tags: ["mysql"]
---

## What is SQL Injection?

SQL injection is an attack where an attacker embeds malicious SQL code in input to trick the database into executing unintended queries.

### Attack Example

```python
# Vulnerable code
username = request.GET['username']  # User input: "'; DROP TABLE users; -- "
sql = f"SELECT * FROM users WHERE username = '{username}'"

# Actually executed SQL:
# SELECT * FROM users WHERE username = ''; DROP TABLE users; -- '
```

### Consequences of Injection

| Consequence | Description |
|------|------|
| Data leakage | Reading sensitive data |
| Data tampering | Modifying or deleting data |
| Privilege escalation | Bypassing authentication |
| Remote commands | Writing Webshell through `INTO OUTFILE` |

---

## 1. Defense Principles

```
Defense Priority:
1. Prepared parameterized queries (most effective)
2. Input validation and filtering
3. Minimum database privileges
4. Database security configuration
```

---

## 2. Prepared Statement

Prepared statements separate SQL structure from parameters. Parameters are treated as data, not SQL code, fundamentally preventing injection.

### Java (JDBC)

```java
// ❌ Unsafe: string concatenation
String sql = "SELECT * FROM users WHERE name = '" + name + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);

// ✅ Safe: prepared statement
String sql = "SELECT * FROM users WHERE name = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, name);
ResultSet rs = pstmt.executeQuery();
```

### Python (PyMySQL)

```python
# ❌ Unsafe
cursor.execute(f"SELECT * FROM users WHERE name = '{name}'")

# ✅ Safe: parameterized query
cursor.execute("SELECT * FROM users WHERE name = %s", (name,))
```

### Node.js (mysql2)

```javascript
// ✅ Safe: placeholders
const sql = 'SELECT * FROM users WHERE name = ?';
connection.query(sql, [name], (err, results) => { ... });
```

### PHP (PDO)

```php
// ✅ Safe: bound parameters
$stmt = $pdo->prepare("SELECT * FROM users WHERE name = :name");
$stmt->execute([':name' => $name]);
```

### Go (database/sql)

```go
// ✅ Safe
rows, err := db.Query("SELECT * FROM users WHERE name = ?", name)
```

---

## 3. ORM Protection

ORM frameworks typically use parameterized queries by default:

```python
# Django ORM (safe)
User.objects.filter(name=name)

# SQLAlchemy (safe)
session.query(User).filter(User.name == name).all()
```

```javascript
// Sequelize (safe)
User.findAll({ where: { name: name } });

// Prisma (safe)
prisma.user.findMany({ where: { name: name } });
```

```java
// MyBatis (safe)
@Select("SELECT * FROM users WHERE name = #{name}")
User findByName(String name);
```

**But ORMs also have pitfalls**:

```python
# ❌ Dangerous! Raw SQL query
User.objects.raw(f"SELECT * FROM users WHERE name = '{name}'")
```

```javascript
// ❌ Dangerous! Sequelize raw query
sequelize.query(`SELECT * FROM users WHERE name = '${name}'`);
```

When using raw queries with ORMs, always use parameterized approaches.

---

## 4. Other Injection Point Defenses

### LIKE Queries

```java
// Special handling for % in LIKE
name = name.replace("!", "!!")
         .replace("%", "!%")
         .replace("_", "!_");
String sql = "SELECT * FROM users WHERE name LIKE ? ESCAPE '!'";
// Note: prepared statements still protect SQL structure, ESCAPE handles search logic
```

### IN Queries

```java
// Dynamic IN lists need placeholders expanded
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

### ORDER BY and Table Names

ORDER BY and table names cannot be parameterized; use whitelist validation:

```java
// ❌ Unsafe
String sql = "SELECT * FROM users ORDER BY " + sortColumn;

// ✅ Safe: whitelist validation
List<String> validColumns = Arrays.asList("id", "name", "created_at");
if (!validColumns.contains(sortColumn)) {
    sortColumn = "id";  // Default value
}
String sql = "SELECT * FROM users ORDER BY " + sortColumn;
// Note: sortColumn has been whitelist-validated, not direct user input
```

---

## 5. Database-Level Security Configuration

### Principle of Least Privilege

```sql
-- ❌ Dangerous: granting DDL privileges to application account
GRANT ALL PRIVILEGES ON shop.* TO 'app'@'%';

-- ✅ Safe: only DML privileges
GRANT SELECT, INSERT, UPDATE, DELETE ON shop.* TO 'app'@'%';

-- ✅ Safer: only the tables that need to be operated
GRANT SELECT, INSERT, UPDATE ON shop.orders TO 'app'@'%';
```

### FILE Privilege

```sql
-- ❌ Never grant FILE privilege to application accounts
-- FILE allows SELECT ... INTO OUTFILE to write files to the server

-- Check which users have FILE privilege
SELECT User, Host FROM mysql.user
WHERE File_priv = 'Y';
```

### local_infile

```ini
# my.cnf
[mysqld]
local-infile = 0
```

Disable `LOAD DATA LOCAL INFILE` to prevent attackers from reading server files.

---

## 6. Other Security Practices

### 1. Input Validation

```java
// Numeric types
int id = Integer.parseInt(request.getParameter("id"));  // Strict type conversion

// Strings
String email = request.getParameter("email");
if (!email.matches("^[\\w.+-]+@[\\w-]+\\.[\\w.]+$")) {
    throw new IllegalArgumentException("Invalid email");
}
```

### 2. Output Encoding

```python
# HTML-escape user input when displaying on web pages
import html
safe_output = html.escape(user_input)
```

### 3. Hide Database Errors

```python
# ❌ Dangerous: exposing database errors directly to users
except MySQLdb.Error as e:
    return f"Database error: {e}"

# ✅ Safe: log errors, return generic message
except MySQLdb.Error as e:
    logger.error(f"Database error: {e}")
    return "An error occurred. Please try again later."
```

---

## 7. SQL Injection Detection

### Automated Detection

```bash
# Use sqlmap for penetration testing (authorization only)
sqlmap -u "http://example.com/page?id=1" --batch

# Static code scanning (Semgrep example)
semgrep --config=p/security-audit --language=python .
```

### Code Review

Check for these patterns:

```python
# 🔴 Dangerous patterns (focus on these during code review):
cursor.execute(f"SELECT ... WHERE name = '{name}'")   # f-string concatenation
cursor.execute("SELECT ... WHERE name = '" + name + "'")  # String concatenation
cursor.execute("SELECT ... WHERE name = %s" % name)       # % formatting
```

---

## 8. Understanding How SQL Injection Works

```python
# Assume login verification SQL:
sql = f"SELECT * FROM users WHERE name = '{username}' AND password = '{password}'"

# Attacker input:
username = "admin"
password = "' OR '1'='1"
# Actually executed SQL:
# SELECT * FROM users WHERE name = 'admin' AND password = '' OR '1'='1'
# '1'='1' is always true, bypassing authentication!
```

Another classic injection:

```python
username = "admin' -- "
# SELECT * FROM users WHERE name = 'admin' -- ' AND password = 'xxx'
# -- is an SQL comment, the following conditions are commented out
```

---

## 9. Secure Connection Information

```python
# ❌ Dangerous: hardcoded database password
DB_PASSWORD = "password123"

# ✅ Safe: use environment variables
import os
DB_PASSWORD = os.getenv("DB_PASSWORD")

# Or use a dedicated secret management service
# AWS Secrets Manager / HashiCorp Vault
```

### Configuration File Security

```bash
# Do not commit .env files to version control
echo ".env" >> .gitignore

# Set minimum permissions for config files
chmod 600 config/database.yml
```

---

## 10. Security Checklist

```
SQL Injection Defense Checklist:
□ All SQL uses parameterized queries or prepared statements
□ Avoid raw SQL concatenation when using ORMs
□ Correctly handle LIKE/IN/ORDER BY special cases
□ Application accounts have only minimum necessary privileges
□ Prohibit FILE privilege
□ Disable local_infile
□ Database error information not exposed to users
□ Sensitive config uses environment variables or secret management
□ Regular code review scanning for injection risks
□ Penetration testing to verify injection defenses
```

---

## Summary

SQL injection is one of the most classic web security vulnerabilities. The key to defense is not how strict the filtering is, but **fundamentally separating data from code** -- parameterized queries are the best practice. Combined with least privilege and security configuration, this effectively defends against the vast majority of injection attacks.

---

