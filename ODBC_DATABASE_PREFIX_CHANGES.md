# ODBC 数据库名前缀功能改造说明

## 概述

本次改造为 OpenSIPS 的 ODBC 模块添加了自动在表名前加上数据库名的功能。所有 SQL 查询将自动使用 `database.table` 格式。

## 修改的文件

### 1. modules/db_unixodbc/db_con.h
- 在 `struct my_con` 中添加了 `database_name` 字段用于存储数据库名
- 添加了 `CON_DATABASE(db_con)` 宏用于访问数据库名

### 2. modules/db_unixodbc/db_con.c
- 在 `db_unixodbc_new_connection()` 中添加了获取数据库名的逻辑
  - 使用 `SQLGetConnectAttr()` 和 `SQL_ATTR_CURRENT_CATALOG` 获取当前数据库名
  - 将数据库名保存到连接结构中
- 在 `db_unixodbc_free_connection()` 中添加了释放数据库名内存的代码

### 3. db/db_query.c
- 添加了 `db_print_table()` 辅助函数
  - 检查连接是否有数据库名
  - 如果有，打印 `database.table` 格式
  - 如果没有，只打印表名
- 修改了所有 SQL 拼接函数以使用 `db_print_table()`：
  - `db_do_query()` - SELECT 查询
  - `db_do_insert()` - INSERT 语句
  - `db_do_delete()` - DELETE 语句
  - `db_do_update()` - UPDATE 语句
  - `db_do_replace()` - REPLACE 语句

## 工作原理

### 1. 连接建立时
```c
// 在 SQLDriverConnect 成功后
SQLGetConnectAttr(ptr->dbc, SQL_ATTR_CURRENT_CATALOG, 
                  db_name, sizeof(db_name), &db_name_len);
// 保存数据库名到 ptr->database_name
```

### 2. SQL 拼接时
```c
// 原来的方式
snprintf(sql_buf, len, "select * from %.*s", table->len, table->s);
// 结果: select * from users

// 新的方式
db_print_table(_h, sql_buf + off, len - off);
// 结果: select * from mydb.users (如果 database_name = "mydb")
```

## 示例

### odbc.ini 配置
```ini
[my_dsn]
Description = MySQL Database
Driver      = MySQL
Server      = localhost
Database    = opensips_db
Port        = 3306
```

### OpenSIPS 配置
```
modparam("usrloc", "db_url", "unixodbc://user:pass@localhost/my_dsn")
```

### 生成的 SQL
**修改前：**
```sql
SELECT * FROM location WHERE username='alice'
INSERT INTO location (username, contact) VALUES ('alice', 'sip:alice@example.com')
UPDATE location SET contact='sip:alice@new.com' WHERE username='alice'
DELETE FROM location WHERE username='alice'
```

**修改后：**
```sql
SELECT * FROM opensips_db.location WHERE username='alice'
INSERT INTO opensips_db.location (username, contact) VALUES ('alice', 'sip:alice@example.com')
UPDATE opensips_db.location SET contact='sip:alice@new.com' WHERE username='alice'
DELETE FROM opensips_db.location WHERE username='alice'
```

## 兼容性

- 如果 ODBC 驱动不支持 `SQL_ATTR_CURRENT_CATALOG`，将回退到不带数据库名前缀的方式
- 对于非 ODBC 数据库模块（MySQL、PostgreSQL 等），不受影响
- 向后兼容：如果无法获取数据库名，行为与之前相同

## 编译

```bash
make modules=modules/db_unixodbc modules
```

## 测试建议

1. 验证连接建立时是否正确获取数据库名
   - 查看日志：`database name from ODBC: xxx`

2. 验证 SQL 语句格式
   - 启用 SQL 日志查看生成的查询语句

3. 测试各种操作
   - 用户注册（INSERT）
   - 用户查询（SELECT）
   - 用户更新（UPDATE）
   - 用户删除（DELETE）

## 注意事项

1. 确保 ODBC 驱动支持 `SQL_ATTR_CURRENT_CATALOG` 属性
2. 数据库名会在连接建立时获取并缓存，不会动态更新
3. 如果需要切换数据库，需要重新建立连接
