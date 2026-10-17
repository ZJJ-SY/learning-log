# upload-labs 文件上传靶场笔记

> 靶场：upload-labs（共 21 关）
> 环境：Windows 11 + phpStudy Pro / Apache 2.4.39 + **mod_fcgid** + PHP 5.2.17
> 入口：`http://localhost/upload-labs-master/`

---

## 笔记列表

| 文档 | 内容 |
|---|---|
| [**.htaccess 上传绕过与 PHP 环境探测**](htaccess上传绕过与PHP环境探测.md) | Pass-04 的完整解法。含三种部署方式（mod_php / mod_fcgid / PHP-FPM）对应的正确写法、排查决策树、五种踩坑、路径探测六法、防御方案 |

---

## 本靶场的环境要点

这些是**本机特有**的结论，直接照抄网上的 writeup 会踩坑：

| 项目 | 值 | 影响 |
|---|---|---|
| Web 服务器 | **Apache 2.4.39**（不是 nginx） | `.htaccess` 才能生效 |
| PHP 运行方式 | **mod_fcgid**（FastCGI，不是 mod_php） | `AddType application/x-httpd-php` **无效**，必须用 `AddHandler` + `FcgidWrapper` |
| upload-labs 用的 PHP | **5.2.17**（单独指定） | Pass-12/13 的 `%00` 截断需要 PHP < 5.3.4 |
| 扩展名映射 | `.php .php3 .php4 .php5 .php7 .phtml .pht .PHP .Php` | Pass-03/04/05/06 靠这些绕过黑名单 |

### ⚠️ 最容易踩的一个坑

**`AddHandler` 是大小写不敏感的，`FcgidWrapper` 是大小写敏感的。**

两者对不上就返回 **500**，而且错误日志里一个字都不留。

```apache
AddHandler fcgid-script .jpg     # .JPG 也能匹配上
FcgidWrapper "…php-cgi.exe" .jpg # 但 .JPG 找不到 wrapper → 500
```

同样的坑在 **Pass-06**（上传 `.PHP`）也出现过。详见笔记的「坑 5」。

---

## 辅助工具

靶场目录下有两个自建的辅助文件（不属于关卡）：

| 文件 | 作用 |
|---|---|
| `_upload_viewer.php` | 可视化查看 `upload/` 目录，按时间倒序、高亮刚上传的文件、标注扩展名能否执行 |
| `_env_check.php` | 环境自检：PHP 版本、扩展、GD/exif 是否可用 |

---

*笔记内容基于本机靶场逐项实测编写。*