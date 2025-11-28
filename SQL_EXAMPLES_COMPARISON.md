# SQL 语句修改前后对比

## 1. SELECT 查询

### 简单查询
**修改前：**
```sql
SELECT * FROM location
```

**修改后：**
```sql
SELECT * FROM "opensips_db"."location"
```

### 指定字段查询
**修改前：**
```sql
SELECT username, contact, expires FROM location
```

**修改后：**
```sql
SELECT "username", "contact", "expires" FROM "opensips_db"."location"
```

### 带 WHERE 条件
**修改前：**
```sql
SELECT username, contact FROM location WHERE username='alice'
```

**修改后：**
```sql
SELECT "username", "contact" FROM "opensips_db"."location" WHERE "username"='alice'
```

### 带多个 WHERE 条件
**修改前：**
```sql
SELECT * FROM location WHERE username='alice' AND domain='example.com'
```

**修改后：**
```sql
SELECT * FROM "opensips_db"."location" WHERE "username"='alice' AND "domain"='example.com'
```

### 带 OR 条件
**修改前：**
```sql
SELECT * FROM location WHERE username='alice' OR username='bob'
```

**修改后：**
```sql
SELECT * FROM "opensips_db"."location" WHERE "username"='alice' OR "username"='bob'
```

### 带 ORDER BY
**修改前：**
```sql
SELECT username, expires FROM location WHERE domain='example.com' ORDER BY expires
```

**修改后：**
```sql
SELECT "username", "expires" FROM "opensips_db"."location" WHERE "domain"='example.com' ORDER BY "expires"
```

### 带操作符
**修改前：**
```sql
SELECT * FROM location WHERE expires>1234567890
```

**修改后：**
```sql
SELECT * FROM "opensips_db"."location" WHERE "expires">1234567890
```

## 2. INSERT 语句

### 单行插入
**修改前：**
```sql
INSERT INTO location (username, contact, expires) VALUES ('alice', 'sip:alice@example.com', 3600)
```

**修改后：**
```sql
INSERT INTO "opensips_db"."location" ("username", "contact", "expires") VALUES ('alice', 'sip:alice@example.com', 3600)
```

### 批量插入（如果支持）
**修改前：**
```sql
INSERT INTO location (username, contact) VALUES ('alice', 'sip:alice@example.com'), ('bob', 'sip:bob@example.com')
```

**修改后：**
```sql
INSERT INTO "opensips_db"."location" ("username", "contact") VALUES ('alice', 'sip:alice@example.com'), ('bob', 'sip:bob@example.com')
```

## 3. UPDATE 语句

### 简单更新
**修改前：**
```sql
UPDATE location SET contact='sip:alice@new.com'
```

**修改后：**
```sql
UPDATE "opensips_db"."location" SET "contact"='sip:alice@new.com'
```

### 带 WHERE 条件的更新
**修改前：**
```sql
UPDATE location SET contact='sip:alice@new.com', expires=7200 WHERE username='alice'
```

**修改后：**
```sql
UPDATE "opensips_db"."location" SET "contact"='sip:alice@new.com', "expires"=7200 WHERE "username"='alice'
```

### 多条件更新
**修改前：**
```sql
UPDATE location SET expires=3600 WHERE username='alice' AND domain='example.com'
```

**修改后：**
```sql
UPDATE "opensips_db"."location" SET "expires"=3600 WHERE "username"='alice' AND "domain"='example.com'
```

## 4. DELETE 语句

### 简单删除
**修改前：**
```sql
DELETE FROM location
```

**修改后：**
```sql
DELETE FROM "opensips_db"."location"
```

### 带 WHERE 条件的删除
**修改前：**
```sql
DELETE FROM location WHERE username='alice'
```

**修改后：**
```sql
DELETE FROM "opensips_db"."location" WHERE "username"='alice'
```

### 多条件删除
**修改前：**
```sql
DELETE FROM location WHERE username='alice' AND expires<1234567890
```

**修改后：**
```sql
DELETE FROM "opensips_db"."location" WHERE "username"='alice' AND "expires"<1234567890
```

## 5. REPLACE 语句

### 简单替换
**修改前：**
```sql
REPLACE INTO location (username, contact) VALUES ('alice', 'sip:alice@example.com')
```

**修改后：**
```sql
REPLACE INTO "opensips_db"."location" ("username", "contact") VALUES ('alice', 'sip:alice@example.com')
```

## 6. 特殊场景

### 大小写敏感字段名
**修改前：**
```sql
SELECT UserName, ContactInfo FROM Location WHERE UserName='Alice'
```

**修改后：**
```sql
SELECT "UserName", "ContactInfo" FROM "opensips_db"."Location" WHERE "UserName"='Alice'
```
✅ 保留了原始大小写

### 包含保留字的字段名
**修改前（可能失败）：**
```sql
SELECT user, order FROM table WHERE user='alice'
```

**修改后（正常工作）：**
```sql
SELECT "user", "order" FROM "opensips_db"."table" WHERE "user"='alice'
```
✅ 保留字被正确引用

### NULL 值处理
**修改前：**
```sql
SELECT * FROM location WHERE contact=NULL
```

**修改后：**
```sql
SELECT * FROM "opensips_db"."location" WHERE "contact"=NULL
```
✅ NULL 值不需要引号

### 数字类型
**修改前：**
```sql
INSERT INTO location (username, expires, cseq) VALUES ('alice', 3600, 1)
```

**修改后：**
```sql
INSERT INTO "opensips_db"."location" ("username", "expires", "cseq") VALUES ('alice', 3600, 1)
```
✅ 数字值不需要引号

## 7. 复杂查询示例

### 用户注册场景
**修改前：**
```sql
INSERT INTO location (username, domain, contact, expires, q, callid, cseq, flags, cflags, user_agent, received, path, socket, methods, last_modified, sip_instance, attr) 
VALUES ('alice', 'example.com', 'sip:alice@192.168.1.100:5060', 1234567890, -1.0, 'abc123@192.168.1.100', 1, 0, 0, 'SIP Client/1.0', 'sip:alice@192.168.1.100:5060', NULL, 'udp:192.168.1.1:5060', 8191, '2024-01-01 12:00:00', NULL, NULL)
```

**修改后：**
```sql
INSERT INTO "opensips_db"."location" ("username", "domain", "contact", "expires", "q", "callid", "cseq", "flags", "cflags", "user_agent", "received", "path", "socket", "methods", "last_modified", "sip_instance", "attr") 
VALUES ('alice', 'example.com', 'sip:alice@192.168.1.100:5060', 1234567890, -1.0, 'abc123@192.168.1.100', 1, 0, 0, 'SIP Client/1.0', 'sip:alice@192.168.1.100:5060', NULL, 'udp:192.168.1.1:5060', 8191, '2024-01-01 12:00:00', NULL, NULL)
```

### 用户查询场景
**修改前：**
```sql
SELECT contact, expires, q, callid, cseq FROM location WHERE username='alice' AND domain='example.com' ORDER BY q
```

**修改后：**
```sql
SELECT "contact", "expires", "q", "callid", "cseq" FROM "opensips_db"."location" WHERE "username"='alice' AND "domain"='example.com' ORDER BY "q"
```

### 过期记录清理场景
**修改前：**
```sql
DELETE FROM location WHERE expires<1234567890
```

**修改后：**
```sql
DELETE FROM "opensips_db"."location" WHERE "expires"<1234567890
```

## 8. 标识符引用规则总结

| 元素类型 | 是否加引号 | 示例 |
|---------|----------|------|
| 数据库名 | ✅ 是 | `"opensips_db"` |
| 表名 | ✅ 是 | `"location"` |
| 字段名 | ✅ 是 | `"username"` |
| 字符串值 | ❌ 否（用单引号） | `'alice'` |
| 数字值 | ❌ 否 | `3600` |
| NULL 值 | ❌ 否 | `NULL` |
| 操作符 | ❌ 否 | `=`, `>`, `<`, `AND`, `OR` |
| 关键字 | ❌ 否 | `SELECT`, `FROM`, `WHERE`, `ORDER BY` |

## 9. 数据库兼容性说明

### PostgreSQL
```sql
-- 完全兼容，无需额外配置
SELECT "username" FROM "opensips_db"."location"
```

### MySQL
```sql
-- 需要启用 ANSI_QUOTES 模式
SET sql_mode='ANSI_QUOTES';
SELECT "username" FROM "opensips_db"."location"

-- 或者在 odbc.ini 中配置
[my_dsn]
Option = 3
```

### SQL Server
```sql
-- 需要启用 QUOTED_IDENTIFIER
SET QUOTED_IDENTIFIER ON;
SELECT "username" FROM "opensips_db"."location"
```

### Oracle
```sql
-- 完全兼容，无需额外配置
SELECT "username" FROM "opensips_db"."location"
```

## 10. 测试建议

### 基本功能测试
```bash
# 1. 启用 SQL 日志
# 在 opensips.cfg 中设置
debug=4

# 2. 查看生成的 SQL
tail -f /var/log/opensips.log | grep -i "select\|insert\|update\|delete"

# 3. 验证双引号是否正确添加
# 应该看到类似：
# SELECT "username", "contact" FROM "opensips_db"."location" WHERE "username"='alice'
```

### 压力测试
```bash
# 使用 SIPp 进行注册压力测试
sipp -sn uac -s alice@example.com -r 100 -m 10000 192.168.1.1

# 监控数据库查询
# 确保所有查询都正确使用了双引号标识符
```
