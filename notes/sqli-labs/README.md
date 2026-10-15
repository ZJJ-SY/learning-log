# sqli-labs Less-1 ~ Less-20 通关笔记

> 靶场：sqli-labs-master（本地授权环境）
> 环境：Windows 11 + phpStudy Pro（Apache 2.4.39 / PHP 5.2.17nts / MySQL 5.7.26）
> 入口：`http://localhost/sqli-labs-master/`
>
> 本系列所有 payload **均在本地靶场实测通过**，文中的输出都是真实抓取结果，不是抄来的。

---

## 怎么用这套笔记

1. 先读 **[基础知识](00-基础知识.md)** —— 注释符、判断列数、找显示位、常用函数、各关成功信号对照表，以及**本环境的 3 个特殊结论**。
2. 再按顺序做 **[Less-1](Less-01.md)** → **[Less-20](Less-20.md)**，每关都是独立文件。
3. 想一次看完 / 打印，用 **[完整版合集](Less-1-20-全合集.md)**。

---

## 关卡总览

| 关卡 | 注入类型 | 闭合方式 | 一句话 |
|---|---|---|---|
| [Less-1](Less-01.md) | GET · 报错型 | `'` | 最经典的字符型注入 |
| [Less-2](Less-02.md) | GET · 报错型 | `无（数字型）` | 不用闭合引号 |
| [Less-3](Less-03.md) | GET · 报错型 | `')` | 引号外还套了括号 |
| [Less-4](Less-04.md) | GET · 报错型 | `")` | 双引号 + 括号 |
| [Less-5](Less-05.md) | GET · 报错盲注 | `'` | 不回显数据，用报错函数 |
| [Less-6](Less-06.md) | GET · 报错盲注 | `"` | 同上，双引号闭合 |
| [Less-7](Less-07.md) | GET · 导出文件 | `'))` | 本环境 outfile 被禁，改用盲注 |
| [Less-8](Less-08.md) | GET · 布尔盲注 | `'` | 只看「You are in」在不在 |
| [Less-9](Less-09.md) | GET · 时间盲注 | `'` | 只看响应时间 |
| [Less-10](Less-10.md) | GET · 时间盲注 | `"` | 同上，双引号 |
| [Less-11](Less-11.md) | POST · 报错型 | `'` | 登录框注入 |
| [Less-12](Less-12.md) | POST · 报错型 | `")` | 登录框，双引号加括号 |
| [Less-13](Less-13.md) | POST · 盲注 | `')` | 不显示数据，只给图片 |
| [Less-14](Less-14.md) | POST · 盲注 | `"` | 同上 |
| [Less-15](Less-15.md) | POST · 盲注 | `'` | 布尔 + 时间盲注 |
| [Less-16](Less-16.md) | POST · 盲注 | `")` | 同上 |
| [Less-17](Less-17.md) | POST · UPDATE 注入 | `'` | 改密码处注入（会真的改库） |
| [Less-18](Less-18.md) | 请求头注入 · User-Agent | `'` | User-Agent 写库 |
| [Less-19](Less-19.md) | 请求头注入 · Referer | `'` | Referer 写库 |
| [Less-20](Less-20.md) | Cookie 注入 | `'` | Cookie 里的 uname |

---

## 环境说明

| 项目 | 值 |
|---|---|
| Web 服务 | Apache 2.4.39（phpStudy Pro） |
| PHP | 5.2.17nts（原生 `ext/mysql`） |
| MySQL | 5.7.26 |
| `magic_quotes_gpc` | `Off`（**必须**，否则注入的引号会被转义） |
| `secure_file_priv` | `NULL`（**文件读写被禁用**，影响 Less-7） |
| 数据目录 | `E:\phpstudy\phpstudy_pro\Extensions\MySQL5.7.26\data\` |

### ⚠️ 本环境的 3 个特殊结论

1. **`secure_file_priv = NULL`** → MySQL 完全禁止文件导入导出，**Less-7 的 `INTO OUTFILE` 写马走不通**（已实测：文件不会生成）。笔记里给了替代打法。
2. **`magic_quotes_gpc` 必须是 `Off`** → 如果发现所有 payload 都返回同样的结果，八成是它被打开了。改 `php5.2.17nts\php.ini` 后重启服务。
3. **Less-17 会真的改数据库** → 它本质是一条 `UPDATE` 语句。做完记得把密码改回来：`uname=admin&passwd=admin`。

---

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

---

| 类型 | 特征 | 打法 | 关卡 |
|---|---|---|---|
| 联合查询 | 页面有回显位 | `union select` | 1,2,3,4,11,12,20 |
| 报错注入 | 页面回显 SQL 报错 | `extractvalue` / `updatexml` | 5,6,17,18,19,20 |
| 布尔盲注 | 只有 true/false 两种页面 | `and 条件` | 7,8,13,14,15,16 |
| 时间盲注 | 页面完全一样 | `if(条件,sleep(3),0)` | 9,10,15,16 |
| 文件写入 | 能写文件 | `into outfile` | 7（本环境被禁） |

---

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
| **表单里 `--+` 失灵，报错 `near 'x'`** | 浏览器把输入框里字面的 `+` 转义成 `%2B`；而 MySQL 的 `--` 注释要求后面跟空白字符，`--+` 不算注释 | **POST 关一律用 `#`**，或写 `-- qwe`（详见第零章「注释符」） |
| curl 的 `--data-urlencode` 让 `--+` 失效 | `+` 被编码成 `%2B` | 改用 `--data-raw` 自己控制编码 |
| 改完 Cookie 刷新又变回去了 | 点了提交按钮走了 POST 分支 | 只按 F5 刷新 |

---

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

---

*本系列基于本地授权靶场实测编写，所有 payload 均已验证。仅用于安全学习，禁止用于未授权测试。*