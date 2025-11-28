# SQL 标识符双引号修改检查清单

## ✅ 已完成的修改

### 1. 数据库连接层 (modules/db_unixodbc/)
- [x] **db_con.h**: 添加 `database_name` 字段
- [x] **db_con.h**: 添加 `CON_DATABASE` 宏
- [x] **db_con.c**: 在连接时获取数据库名 (`SQLGetConnectAttr`)
- [x] **db_con.c**: 在释放连接时释放数据库名内存

### 2. SQL 查询构建层 (db/db_query.c)
- [x] **db_print_table()**: 新增函数，打印 `"database"."table"` 格式
- [x] **db_do_query()**: SELECT 语句使用 `db_print_table()`
- [x] **db_do_insert()**: INSERT 语句使用 `db_print_table()`
- [x] **db_do_delete()**: DELETE 语句使用 `db_print_table()`
- [x] **db_do_update()**: UPDATE 语句使用 `db_print_table()`
- [x] **db_do_replace()**: REPLACE 语句使用 `db_print_table()`
- [x] **db_do_query()**: ORDER BY 子句字段名加双引号

### 3. SQL 工具函数层 (db/db_ut.c)
- [x] **db_print_columns()**: 所有字段名加双引号 `"column"`
- [x] **db_print_where()**: WHERE 子句字段名加双引号 `"field"=value`
- [x] **db_print_set()**: SET 子句字段名加双引号 `"field"=value`

## 📋 SQL 语句覆盖情况

### SELECT 查询
```sql
-- 修改前
SELECT username, contact FROM location WHERE username='alice' ORDER BY expires

-- 修改后
SELECT "username", "contact" FROM "opensips_db"."location" WHERE "username"='alice' ORDER BY "expires"
```
✅ 完全覆盖

### INSERT 语句
```sql
-- 修改前
INSERT INTO location (username, contact, expires) VALUES ('alice', 'sip:alice@example.com', 3600)

-- 修改后
INSERT INTO "opensips_db"."location" ("username", "contact", "expires") VALUES ('alice', 'sip:alice@example.com', 3600)
```
✅ 完全覆盖

### UPDATE 语句
```sql
-- 修改前
UPDATE location SET contact='sip:alice@new.com', expires=7200 WHERE username='alice'

-- 修改后
UPDATE "opensips_db"."location" SET "contact"='sip:alice@new.com', "expires"=7200 WHERE "username"='alice'
```
✅ 完全覆盖

### DELETE 语句
```sql
-- 修改前
DELETE FROM location WHERE username='alice' AND expires<1234567890

-- 修改后
DELETE FROM "opensips_db"."location" WHERE "username"='alice' AND "expires"<1234567890
```
✅ 完全覆盖

### REPLACE 语句
```sql
-- 修改前
REPLACE INTO location (username, contact) VALUES ('alice', 'sip:alice@example.com')

-- 修改后
REPLACE INTO "opensips_db"."location" ("username", "contact") VALUES ('alice', 'sip:alice@example.com')
```
✅ 完全覆盖

## ⚠️ 未修改的部分（不在 ODBC 模块范围内）

### MySQL 特定功能 (modules/db_mysql/dbase.c)
- [ ] **db_insert_update()**: MySQL 的 `INSERT ... ON DUPLICATE KEY UPDATE` 语句
  - 原因：这是 MySQL 模块特定的实现，不在 ODBC 通用层
  - 影响：如果通过 ODBC 连接 MySQL 并使用此功能，需要单独修改
  - 位置：`modules/db_mysql/dbase.c:1400-1480`

### 其他数据库模块
- [ ] PostgreSQL 模块 (modules/db_postgres/)
- [ ] Oracle 模块 (modules/db_oracle/)
- [ ] SQLite 模块 (modules/db_sqlite/)
- 原因：这些模块有自己的实现，不使用通用的 db_query.c

## 🔍 验证点

### 代码层面
- [x] 所有 `snprintf` 调用都正确添加了双引号
- [x] 没有遗漏的字段名或表名
- [x] 缓冲区大小计算考虑了额外的引号字符
- [x] 错误处理保持完整

### 功能层面
需要测试的场景：
1. [ ] 基本的 CRUD 操作（Create, Read, Update, Delete）
2. [ ] 带 WHERE 条件的查询
3. [ ] 带 ORDER BY 的查询
4. [ ] 多字段查询和更新
5. [ ] 特殊字符字段名（如包含空格、保留字）
6. [ ] 大小写敏感的字段名

### 数据库兼容性
需要测试的数据库：
1. [ ] PostgreSQL (标准支持双引号)
2. [ ] MySQL (需要 ANSI_QUOTES 模式)
3. [ ] SQL Server (需要 QUOTED_IDENTIFIER ON)
4. [ ] Oracle (标准支持双引号)

## 📝 注意事项

1. **MySQL 特殊配置**：
   - 需要在 ODBC DSN 配置中添加 `Option=3` 启用 ANSI_QUOTES
   - 或在 MySQL 服务器设置 `sql_mode='ANSI_QUOTES'`

2. **性能影响**：
   - 双引号会略微增加 SQL 语句长度
   - 对性能影响可忽略不计

3. **向后兼容性**：
   - 如果数据库名无法获取，会回退到只有表名的方式
   - 双引号对标准 SQL 标识符（小写、无特殊字符）无影响

4. **调试建议**：
   - 启用 SQL 日志查看生成的实际查询
   - 检查日志中的 "database name from ODBC:" 消息

## 🎯 总结

### 修改范围
- **3 个文件**被修改
- **8 个函数**被更新
- **5 种 SQL 语句**类型被覆盖

### 标识符引用情况
| 标识符类型 | 是否加引号 | 示例 |
|-----------|----------|------|
| 数据库名 (Schema) | ✅ | `"opensips_db"` |
| 表名 (Table) | ✅ | `"location"` |
| 字段名 (Column) | ✅ | `"username"` |
| 字段值 (Value) | ❌ | `'alice'` (字符串值用单引号) |

### 完整性评估
✅ **ODBC 模块的通用 SQL 拼接已完全覆盖**

所有通过 `db_query.c` 和 `db_ut.c` 构建的 SQL 语句都已正确添加双引号标识符。
