# 调试指南 - 数据库名前缀功能

## 问题诊断

根据日志，之前的问题是：
1. ❌ `could not get database name from ODBC connection` - 无法获取数据库名
2. ❌ SQL 中没有数据库前缀：`select "table_version" from "version"`

## 最新修改

### 1. 在通用 db_con_t 结构中添加 schema 字段
**文件：** `db/db_con.h`
```c
typedef struct {
    const str* table;
    db_ps_t* curr_ps;
    struct query_list *ins_list;
    unsigned long tail;
    str url;
    int flags;
    char* schema;  // 新增：数据库 schema 名称
} db_con_t;

#define CON_SCHEMA(cn) ((cn)->schema)  // 新增宏
```

### 2. ODBC 模块设置 schema
**文件：** `modules/db_unixodbc/dbase.c`
```c
db_con_t* db_unixodbc_init(const str* _url)
{
    db_con_t* con = db_do_init(_url, (void*)db_unixodbc_new_connection);
    if (con && CON_CONNECTION(con)) {
        struct my_con* mycon = (struct my_con*)con->tail;
        if (mycon && mycon->database_name) {
            con->schema = mycon->database_name;  // 设置 schema
            LM_DBG("set schema to: %s\n", con->schema);
        }
    }
    return con;
}
```

### 3. SQL 拼接使用 schema
**文件：** `db/db_query.c`
```c
static inline int db_print_table(const db_con_t* _h, char* _b, const int _l)
{
    // ...
    if (CON_SCHEMA(_h) && *(CON_SCHEMA(_h))) {
        // 打印 "database"."table"
        ret = snprintf(_b, _l, "\"%s\".\"%.*s\"", CON_SCHEMA(_h), table->len, table->s);
        LM_DBG("using schema prefix: %s\n", CON_SCHEMA(_h));
        return ret;
    }
    // 否则只打印 "table"
    ret = snprintf(_b, _l, "\"%.*s\"", table->len, table->s);
    return ret;
}
```

## 编译和测试

### 1. 清理并重新编译
```bash
# 清理旧的编译文件
make clean

# 重新编译 ODBC 模块
make modules=modules/db_unixodbc modules

# 或者编译整个项目
make all
```

### 2. 检查编译输出
确保没有编译错误或警告。

### 3. 启动 OpenSIPS 并查看日志
```bash
# 启动 OpenSIPS（调试模式）
opensips -f /etc/opensips/opensips.cfg -D -E

# 或者查看日志文件
tail -f /var/log/opensips.log
```

## 预期的日志输出

### 成功的情况

#### 1. 连接建立时
```
DBG:db_unixodbc:db_unixodbc_new_connection: opening connection: unixodbc://xxxx:xxxx@localhost/dm8
DBG:db_unixodbc:db_unixodbc_new_connection: connection succeeded
DBG:db_unixodbc:db_unixodbc_new_connection: database name from ODBC: SYSDBA  ✅
DBG:db_unixodbc:db_unixodbc_init: set schema to: SYSDBA  ✅
```

#### 2. SQL 查询时
```
DBG:db_query:db_print_table: using schema prefix: SYSDBA  ✅
DBG:db_query:db_do_query: Query= select "table_version" from "SYSDBA"."version" where "table_name"='subscriber'  ✅
```

### 失败的情况（需要修复）

#### 1. 无法获取数据库名
```
DBG:db_unixodbc:db_unixodbc_new_connection: could not get database name from ODBC connection  ❌
```

**可能原因：**
- ODBC 驱动不支持 `SQL_ATTR_CURRENT_CATALOG`
- ODBC 配置中没有设置 Database 参数

**解决方法：**
检查 odbc.ini 配置：
```ini
[dm8]
Description = DM8 Database
Driver = DM8 ODBC DRIVER
Server = localhost
Database = SYSDBA  # 确保这个参数存在
Port = 5236
```

#### 2. SQL 中没有数据库前缀
```
Query= select "table_version" from "version" where "table_name"='subscriber'  ❌
```

**可能原因：**
- schema 字段没有被正确设置
- db_print_table 函数没有被调用

**解决方法：**
1. 确保重新编译了所有修改的文件
2. 检查 `db_unixodbc_init` 中的日志是否显示 "set schema to: xxx"

## 调试步骤

### 步骤 1：验证数据库名获取
在 `modules/db_unixodbc/db_con.c` 的 `db_unixodbc_new_connection` 函数中，应该看到：
```
DBG:db_unixodbc:db_unixodbc_new_connection: database name from ODBC: SYSDBA
```

如果看不到，说明 `SQLGetConnectAttr` 调用失败。

### 步骤 2：验证 schema 设置
在 `modules/db_unixodbc/dbase.c` 的 `db_unixodbc_init` 函数中，应该看到：
```
DBG:db_unixodbc:db_unixodbc_init: set schema to: SYSDBA
```

如果看不到，说明 schema 没有被正确传递。

### 步骤 3：验证 SQL 拼接
在 `db/db_query.c` 的 `db_print_table` 函数中，应该看到：
```
DBG:db_query:db_print_table: using schema prefix: SYSDBA
```

如果看不到，说明 `CON_SCHEMA(_h)` 为空。

### 步骤 4：验证最终 SQL
查看实际执行的 SQL 语句：
```
Query= select "table_version" from "SYSDBA"."version" where "table_name"='subscriber'
```

应该包含 `"SYSDBA"."version"` 格式。

## 手动测试

### 测试 1：检查 ODBC 连接
```bash
# 使用 isql 测试 ODBC 连接
isql -v dm8 username password

# 在 SQL 提示符下测试
SQL> SELECT * FROM SYSDBA.version;
```

### 测试 2：检查 OpenSIPS 数据库操作
```bash
# 启动 OpenSIPS
opensips -f /etc/opensips/opensips.cfg -D -E

# 在另一个终端监控日志
tail -f /var/log/opensips.log | grep -E "database name|schema|Query="

# 触发数据库操作（例如用户注册）
# 观察日志中的 SQL 语句
```

## 常见问题

### Q1: 编译时出现 "schema undeclared" 错误
**A:** 确保 `db/db_con.h` 中添加了 `char* schema;` 字段。

### Q2: 运行时 schema 为 NULL
**A:** 检查以下几点：
1. ODBC 驱动是否支持 `SQL_ATTR_CURRENT_CATALOG`
2. odbc.ini 中是否配置了 Database 参数
3. `db_unixodbc_init` 是否正确设置了 `con->schema`

### Q3: SQL 中仍然没有数据库前缀
**A:** 检查：
1. 是否重新编译了所有修改的文件
2. `db_print_table` 函数是否被正确调用
3. `CON_SCHEMA(_h)` 是否返回正确的值

## 验证清单

- [ ] 编译成功，无错误和警告
- [ ] 日志显示 "database name from ODBC: xxx"
- [ ] 日志显示 "set schema to: xxx"
- [ ] 日志显示 "using schema prefix: xxx"
- [ ] SQL 语句包含 "database"."table" 格式
- [ ] 所有字段名都有双引号
- [ ] 数据库操作正常（SELECT, INSERT, UPDATE, DELETE）

## 下一步

如果所有检查都通过，但仍然有问题，请提供：
1. 完整的启动日志
2. 具体的错误信息
3. odbc.ini 配置内容
4. 使用的数据库类型和版本
