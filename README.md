# VPS 新手安全设置指南：从首次登录到安全基线（Debian / Ubuntu）

> 一台刚开通的 VPS，初始密码明文躺在邮件里，系统补丁是几个月前的，SSH 大门朝整个互联网敞着。开通第一天，全世界的扫描器就在挨个试它的门锁。
>
> 这篇只讲一件事：**到手之后的设置与安全基线**，不讲往里装什么应用。照着做完，这台机器才算真正归你。

**适用系统**：Debian 11+ / Ubuntu 20.04+（apt 系）
**预计耗时**：最小必做路径约 30 分钟，全部做完约 1 小时
**实测环境**：Ubuntu 24.04 LTS（KVM，1 核 / 1G 内存），本地为 Windows 10，2026 年 10 月完整走通全流程。Debian 部分按系统机制编写，暂未完整实测；在其他系统或商家上遇到不一致的地方，欢迎提 [Issue](https://github.com/kungen1008/vps-setup/issues)。

---

## 目录

- [阅读说明](#阅读说明)
- [整体思路：最危险的一步放到最后](#整体思路最危险的一步放到最后)
- [第 0 步：收好「房产证」，认清救命门](#第-0-步收好房产证认清救命门)
- [第 1 步：连上去](#第-1-步连上去)
- [第 2 步：换锁——改初始密码，并实测 VNC](#第-2-步换锁改初始密码并实测-vnc)
- [第 3 步：打补丁——系统更新](#第-3-步打补丁系统更新)
- [第 4 步：验房——配置、IP、虚拟化](#第-4-步验房配置ip虚拟化)
- [第 5 步：建日常账号——别总用 root](#第-5-步建日常账号别总用-root)
- [第 6 步：配钥匙——密钥登录](#第-6-步配钥匙密钥登录)
- [第 7 步：把「安防系统」一次装齐](#第-7-步把安防系统一次装齐)
- [第 8 步：最后锁门——SSH 加固](#第-8-步最后锁门ssh-加固)
- [第 9 步：重启总验收](#第-9-步重启总验收最重要的一步)
- [第 10 步：打快照、确认续费](#第-10-步验收通过再打快照确认续费)
- [完工清单](#完工清单)
- [附：英文名词小词典](#附英文名词小词典)

---

## 阅读说明

**三个约定**

1. 命令里的**中文是占位词**，要换成你自己的内容，比如 `你的IP`、`用户名`、`你当前SSH端口`。原样粘贴会报错。
2. 遇到不认识的英文名词，不用停下来查，照做不受影响。文末有[小词典](#附英文名词小词典)。
3. 每个代码块执行完，**先看清输出对不对**，再做下一步。别一口气全贴。

> [!WARNING]
> 这篇要改 SSH、开防火墙，一步做错，就可能把自己锁在门外。建议先拿一台新机器照着做；已经在跑业务的机器，先备份好重要数据再动手。

**适用范围**：全文以 Debian / Ubuntu 为例。CentOS / Rocky / Alma 等 RHEL 系的包管理是 `dnf`、防火墙是 `firewalld`、SSH 服务名是 `sshd`，改端口还要处理 SELinux，命令对不上，别硬套。

<details>
<summary><b>📦 默认用户镜像（ubuntu / debian 用户，没有 root 直登）须知——点开看</b></summary>

有些云镜像的默认用户就是 `ubuntu` 或 `debian`，没有 root 直接登录。如果你是这种镜像：

- **连接**：第 1 步命令里的 `root` 换成这个默认用户名（商家给了私钥文件的，再加 `-i 私钥路径`）。连上之后先执行一次 `sudo -i` 切到 root，后面所有步骤都在 root 下做。

有两个坑，都是「你以为在改默认用户，其实在改 root」：

- **坑一**：第 2 步的 `passwd` 要写成 `passwd 你的默认用户名`。在 root 下不带参数，改的是 root 自己的密码。给默认用户设好密码后，用它测 VNC。
- **坑二**：第 6 步凡是涉及这个用户的 `.ssh` 目录，都写完整路径 `/home/默认用户名/.ssh/`，别用 `~`。root 壳里的 `~` 是 `/root`，写错地方等于没配。

第 5 步建用户可以跳过，但末尾那段「用 VNC 实测默认用户密码」要照做。跳了它，救命门就是废的。

</details>

---

## 整体思路：最危险的一步放到最后

用房子打个比方：

```
收房 → 换锁 → 验房 → 配钥匙 → 装齐防盗门/报警器/保险（趁还有管理员权限）
     → 最后才锁门 → 拉总闸验收 → 验收过了，再拍照存档
```

**最危险的「锁门」（SSH 加固）放到最后做。**前面所有要管理员权限的活先干完。否则你中途把 root 登录一关，后面每条命令都要跟权限报错搏斗，还更容易把自己锁在门外。

> [!IMPORTANT]
> **一条贯穿全程的纪律：新路没走通之前，永远不关旧窗口。**
> 每改一处关键设置，都另开一个新窗口测试能正常登录，再关旧的。

### 赶时间？最小必做路径

| 步骤 | 内容 | 必做？ |
|---|---|---|
| 0～6 | 收房、换锁、更新、验房、建用户、配密钥 | ✅ 必做 |
| 7.1 | 防火墙 ufw | ✅ 必做 |
| 7.2 | fail2ban | 👍 建议（降噪保安，不装也不影响核心防线） |
| 7.3 | 自动安全更新 | ✅ 必做 |
| 7.4 / 7.5 | swap、BBR | ⭕ 可选 |
| 7.6 | 时区、主机名 | ⭕ 可选 |
| 8.1～8.3 | SSH 加固：纯密钥 + 禁 root | ✅ 必做 |
| 8.4 | 改 SSH 端口 | ⭕ 可选 |
| 9 | 重启验收 | ✅ 必做 |
| 10 | 快照、续费提醒 | ✅ 必做 |

---

## 第 0 步：收好「房产证」，认清救命门

开通邮件和商家面板里有这些东西：

- IP 地址、SSH 端口（多数是 22，有些商家默认是别的端口，记准你自己的）
- root 用户名、初始密码（或初始密钥）
- 商家面板账号
- VNC 控制台入口

**全部存进密码管理器**，别只躺在邮箱里。

**VNC 控制台**是商家面板里的网页版「显示器」，不走 SSH，是最后的救命门。但它有个脾气：VNC 登录用的是系统账号密码，**常常不能粘贴、键盘布局还可能不一样**。一个长随机密码在里面手敲，错一个字符你都看不出来。

所以后面每次设完密码，我们都要实际用 VNC 登录一次确认能进（第 2 步、第 5 步各测一次）。顺便确认商家面板有没有「重置密码 / 救援模式」功能，心里留个底。

---

## 第 1 步：连上去

Windows 打开 PowerShell 或 Windows Terminal，Mac 打开终端，命令一样：

```bash
ssh root@你的IP -p 22
```

> 端口不是 22 的，把 `22` 换成你自己的，后文同理。

- **指纹确认**：第一次连接会问你认不认这台机器的「身份证」。商家如果在面板里公布了主机指纹，花一分钟核对再输 `yes`；没公布的，第一次也只能先信。
- **核对要快**：服务器只给每个新连接约 2 分钟完成登录（`LoginGraceTime`）。核对指纹花太久，输完 `yes` 会看到 `Connection closed by ... port 22`。指纹已经保存了，**直接重连一次**就行。
- **输密码时屏幕上没有任何显示**，不是键盘坏了，输完直接回车。

> [!TIP]
> 以后用新电脑或新客户端（比如 Windows 自带的 `ssh`）第一次连这台机器，也会再问一次指纹。想核对的话，在已登录的服务器窗口执行 `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`，对比输出的 `SHA256:...` 是否一致。

以后重装了系统，再连会跳一个大红警告 `HOST KEY CHANGED`。那是新系统的指纹和旧记录对不上，先清掉旧记录再连：

```bash
ssh-keygen -R 你的IP

# SSH 端口不是 22 的，要带端口写（记录里存的是这种格式，不带端口清不掉）
ssh-keygen -R "[你的IP]:你的端口"
```

### 配个别名，以后省事

在**自己电脑**的 `~/.ssh/config` 里写：

```sshconfig
Host myvps
    HostName 你的IP
    Port 22
    User root
```

之后 `ssh myvps` 就能连。端口、用户后面改了，回来改这两行就行。

> [!WARNING]
> **Windows 用户注意：记事本默认会把文件存成 `config.txt`**，ssh 读不到，会报 `Could not resolve hostname myvps`（中文系统可能显示成一串乱码，意思是“不知道这样的主机”）。
>
> 最稳的办法是在 PowerShell 里用命令直接写文件（`'@` 必须单独一行、顶格）：
>
> ```powershell
> @'
> Host myvps
>     HostName 你的IP
>     Port 22
>     User root
> '@ | Set-Content -Path $env:USERPROFILE\.ssh\config -Encoding ascii
> ```
>
> 已经存成 `config.txt` 的，把扩展名删掉即可。`.ssh` 文件夹不存在的话，先执行 `mkdir $env:USERPROFILE\.ssh`。

> [!IMPORTANT]
> **看提示符，分清你在哪台机器上。**同一个 PowerShell 窗口，`ssh` 登录前后操作的是两台不同的机器：
>
> | 提示符 | 你在哪里 |
> |---|---|
> | `PS C:\Users\你的用户名>` | 自己的 Windows 电脑 |
> | `用户名@主机名:~$` | 服务器（普通用户） |
> | `root@主机名:~#` | 服务器（root） |
>
> 本文会标明命令在哪边执行。建议开两个窗口：一个 `ssh` 连进服务器，一个留在本地。

### 用图形客户端也一样

不喜欢命令行的，用 Termius、FinalShell、Xshell 这类图形客户端也一样。填 IP、端口、用户名、密码就能连，后面的操作在它们自带的终端里做，没有区别。到时回来改两处：

- **第 6 步配密钥时**：这类客户端一般有「导入私钥」入口，把生成的私钥文件指给它。个别客户端对带口令的私钥支持不一，导入不了的话，直接在客户端里新建一对密钥，再把它的公钥按第 6 步的方法追加到服务器。
- **第 8 步改了端口的**：把连接配置里的端口号换成新的。

---

## 第 2 步：换锁——改初始密码，并实测 VNC

```bash
passwd
```

> 默认用户镜像：这里要写 `passwd 你的默认用户名`，见开头的须知。

初始密码在开通邮件里是明文，经过商家系统、邮件系统好几手，等于半公开。**不改这一步，后面所有设置都是白做。**

密码用密码管理器生成、存好，但要兼顾一件事：**这个密码你得能在 VNC 里手敲进去**。生成时避开容易敲错的符号组合。

改完马上去商家面板，用 VNC 以 root 登录一次（默认用户镜像则用那个默认用户）。现在能手敲进去，第 8、9 步万一要靠它救命时才不会抓瞎。

---

## 第 3 步：打补丁——系统更新

```bash
apt update && apt upgrade -y
```

商家给的系统镜像往往是几个月前做的，已知漏洞全在，先补上。两个小插曲提前知道：

**① 弹出「配置文件被修改过」的对话框**

最常见的是更新 `openssh-server` 时出现的这个：

```
A new version of configuration file /etc/ssh/sshd_config is available,
but the version installed currently has been locally modified.

    install the package maintainer's version
  > keep the local version currently installed
    show the differences between the versions
    ...
```

选 **keep the local version currently installed**（保留当前版本），回车确认。

原因：云镜像大多改过 `sshd_config`（比如端口、登录方式）。选第一项会被官方默认配置覆盖，万一商家改过端口，覆盖后可能就连不上了。本文第 8 步的加固写在单独的文件里，不改主配置，所以这里保留本地版本不会有任何冲突。

**② 新版 Ubuntu 可能弹出全屏界面，问「要重启哪些服务」**：直接回车用默认值。

**③ 装完检查要不要重启**

更新结束时，Ubuntu 往往会在输出末尾提示：

```
Pending kernel upgrade!
Running kernel version:
  6.8.0-31-generic
Diagnostics:
  The currently running kernel version is not the expected kernel version 6.8.0-xxx-generic.
```

意思是新内核装好了，但要重启才能生效。下面这些也会一起出现，都不用处理，重启后自动解决：`os-prober will not be executed`、`Service restarts being deferred`、`User sessions running outdated binaries`。

也可以用命令确认：

```bash
ls /var/run/reboot-required
```

文件存在就 `reboot`，等一两分钟重连，再用 `uname -r` 确认已经换成新内核版本。

> [!NOTE]
> 这个文件是 **Ubuntu** 的机制。Debian 默认不会生成它，所以**在 Debian 上「文件不存在」不代表不用重启**。简单判断：更新列表里出现了 `linux-image` 开头的包（内核更新），就直接 `reboot` 一次。

---

## 第 4 步：验房——配置、IP、虚拟化

### 对配置

```bash
free -h               # 内存
df -h                 # 硬盘
nproc                 # CPU 核数
systemd-detect-virt   # 虚拟化类型
```

最后一条看一眼：

| 输出 | 含义 |
|---|---|
| `kvm` | 正经的独享内核虚拟化 ✅ |
| `openvz` / `lxc` | 「共享内核」型，容易被邻居挤兑，部分功能受限（比如 7.4 的 swap、7.5 的 BBR 可能用不了） |

### 查 IP 的「户口」

精简版系统可能没装 curl，先补上：

```bash
apt install curl -y
curl ipinfo.io
```

返回里看 `city`、`country`、`org`：**归属地和你下单的地区对得上吗？**买的洛杉矶，别发你一台法兰克福的。

再提醒一句：独享 IP 不等于干净 IP。前任租客拿它发过垃圾邮件的话，这个 IP 可能已经进了黑名单。可以去 [Spamhaus](https://check.spamhaus.org) 这类查询站搜一下自己的 IP，有问题趁早要求更换。

机器带 IPv6 的，再记一笔：后面防火墙、安全组、监听检查，都要把 v6 一并算进去。

---

## 第 5 步：建日常账号——别总用 root

root 是这台机器的上帝权限，日常拿着它裸奔，输错一条命令没有后悔药。建一个普通用户当日常入口：

```bash
adduser 你起的用户名
usermod -aG sudo 你起的用户名
```

- `adduser` 会依次问全名、房间号、电话等信息，一路回车跳过即可。
- Debian 精简系统提示没有 sudo 的，先执行 `apt install sudo -y` 再回来。

**这个用户的密码别当摆设**，它身兼两职：

1. 日后每次 `sudo` 提权要输它；
2. SSH 密码登录关掉之后，用 VNC 登录输的也是它。

所以标准和第 2 步一样：足够长，但你能手敲对。

**建完立刻验证两遍：**

- [ ] 新开窗口，用新用户 SSH 登录，确认能进、`sudo` 能提权
- [ ] 用 VNC 以这个用户登录一次，确认密码手敲能进

> [!TIP]
> **默认用户镜像**不用建用户，但有一件事不能省：这类镜像的默认用户通常**压根没有密码**（平时靠免密 sudo 干活），没密码 VNC 就登不上。第 2 步你已经给它设了密码，这里再用 VNC 实测一次。第 5 步对你来说就这一段，不要整步跳过。

---

## 第 6 步：配钥匙——密钥登录

密码再长，也能被「猜」：公网上的机器每天被试几千次密码，这叫暴力破解。

密钥登录是另一套机制：一对钥匙，**私钥**锁在你自己电脑里，永不外传；**公钥**挂到服务器上。登录时两边对暗号，不走密码，就没有「被猜中」这回事。

### 6.1 在自己电脑上生成密钥

```bash
ssh-keygen -t ed25519
```

生成时会问你给私钥设不设口令（passphrase）。**建议设一个。**一路回车等于私钥裸奔，电脑被偷或中木马，钥匙直接被人拿走。

设了口令后，靠 `ssh-agent` 帮你记住已解锁的钥匙，日常只输一次：

**Mac**：在 `~/.ssh/config` 里加：

```sshconfig
Host *
    AddKeysToAgent yes
    UseKeychain yes   # 口令存进系统钥匙串，重启后也免重输
```

**Windows**：ssh-agent 服务默认是禁用的，要用**管理员 PowerShell** 启用一次。按 `Win + X`，选「Windows PowerShell(管理员)」或「终端(管理员)」，窗口标题以「管理员:」开头就对了。执行：

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add
```

`ssh-add` 会让你输一次私钥口令，看到 `Identity added` 就成功了。之后电脑重启 agent 也会自动启动、记住这把钥匙。这几行都在同一个管理员窗口执行即可，做完可以关掉它。

### 6.2 把公钥送上服务器

**Mac / Linux：**

```bash
ssh-copy-id -p 22 用户名@你的IP
```

**Windows PowerShell**（普通窗口即可，效果一样）：

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 22 用户名@你的IP "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

会提示输入这个用户的**账号密码**，输完没有任何输出、直接回到提示符，就是成功了。因为是以该用户身份登录写入的，文件属主自然正确，权限也一步设好，不会遇到 6.3 说的问题。

> [!NOTE]
> 这两条能成功，是因为现在密码登录还开着。等第 8 步关掉密码登录后再想补公钥，就只能先用已有钥匙登录，或者走 VNC。

<details>
<summary><b>📦 默认用户镜像：送钥匙命令会失败，点开看正确做法</b></summary>

这类镜像的 SSH 本来就只认商家预置的密钥，密码登录是关着的，上面两条命令都会直接失败。

正确做法：先用商家给你的那把钥匙登录进去，然后在服务器上把你自己的公钥**整行追加**进：

```
/home/默认用户名/.ssh/authorized_keys
```

如果这时你开着 root 的壳，千万别写 `~/.ssh`，那是 `/root/.ssh`。追加完按下面的方法修正属主和权限。

</details>

### 6.3 手动追加的隐形坑：属主和权限

用 root 身份把公钥写进普通用户家目录，文件属主会变成 root，sshd 会因为属主不对**拒绝这把钥匙**，表现就是「密钥死活不通」。写完补上：

```bash
chown -R 用户名:用户名 /home/用户名/.ssh
chmod 700 /home/用户名/.ssh
chmod 600 /home/用户名/.ssh/authorized_keys
```

### 6.4 测试

新开窗口登录：**不用输账号密码就能进，密钥才算配好。**（私钥设了口令的，这里输的是钥匙口令，被 agent 记住后就不再问。）

> [!CAUTION]
> **密钥没测通，绝不进行后面的锁门步骤。**

### 6.5 两件保命的事

1. **备份私钥**：加密后存进密码管理器或加密 U 盘。关掉密码登录后，私钥就是唯一的门。丢了就是永久锁门，只能靠 VNC 和重置功能续命。
2. **检查预置钥匙**：看有没有你不认识的公钥（有些镜像会预装商家的管理密钥）：

   ```bash
   cat /root/.ssh/authorized_keys                # root 登录的镜像
   cat /home/默认用户名/.ssh/authorized_keys      # 默认用户镜像
   ```

   不认识的先问清商家是什么，再决定留不留。

条件允许的话，给第二台设备（比如手机上的 Termius）也配一把公钥，一把丢了还有一把。

---

## 第 7 步：把「安防系统」一次装齐

下面这几样都要管理员权限，**趁锁门之前全干完**。默认用户镜像先 `sudo -i` 切到 root。

### 7.1 防火墙 ufw

Debian 默认没装，先装上（Ubuntu 一般自带）：

```bash
apt install ufw -y
ufw default deny incoming
ufw default allow outgoing
ufw allow 你当前SSH端口/tcp
ufw enable
ufw status
```

> [!WARNING]
> **放行的是你「当前正在使用」的 SSH 端口，不是写死的 22。**你家端口是别的数字，照抄 22 再一开防火墙，当场就把自己关在门外了。

- `ufw enable` 会弹一句确认（「可能影响现有 SSH 连接，继续吗」），输 `y` 回车，不是卡住了。
- **只放行真正要用的端口。**80、443 只有跑网站时才开。
- 机器有 IPv6 的，确认 `/etc/default/ufw` 里是 `IPV6=yes`（默认就是）。
- 商家面板如果有「安全组」，它是另一层防火墙，两边规则都要对上。

### 7.2 fail2ban：先装上，定位要摆正

```bash
apt install fail2ban -y
```

**Debian 12 / Ubuntu 22.04 以后的新系统，SSH 日志在 journald 里，没有传统的 `/var/log/auth.log`。**fail2ban 默认去读日志文件，读不到就会启动失败，或者装了等于没装。所以先告诉它去读 journald：

```bash
apt install python3-systemd -y

cat > /etc/fail2ban/jail.local <<'EOF'
[sshd]
enabled = true
backend = systemd
EOF

systemctl enable --now fail2ban
systemctl restart fail2ban
sleep 2
fail2ban-client status sshd
```

- 最后一行**必做**：亲眼确认 sshd 这个 jail 真的在跑。
- 加 `sleep 2` 是因为服务刚启动时 jail 可能还没建好，立刻查会报错，别被吓到。
- 还是报错的，用 `journalctl -u fail2ban -n 30` 看具体原因。

> [!NOTE]
> 定位要说清：第 8 步关掉密码登录后，爆破本来就不可能成功。fail2ban 的作用变成挡掉海量扫描、减少日志噪音，外加万一哪天你临时开了密码登录，它还在岗。**它是保安，不是防盗门本体。**
>
> 端口和白名单先不动，等第 8 步 SSH 端口定了，再到 8.5 一起收尾。

### 7.3 自动安全更新

```bash
apt install unattended-upgrades -y
dpkg-reconfigure --priority=low unattended-upgrades
```

选 **Yes**，以后安全补丁自动安装。例外：**内核更新装完不会自动重启生效**，该手动 `reboot` 时还是要重启（判断方法见第 3 步）。

### 7.4 小内存机器加 swap（可选）

内存 1G 及以下的机器，加一块交换空间兜底，免得内存一爆直接死机。

**先检查是不是已经有了**，不少商家会预先配好 swap 分区或 swap 文件：

```bash
swapon --show
```

- **有输出**（比如 `/dev/vda3 partition 1G`）：已经有 swap，**整步跳过**。再加一个只会白占硬盘；
- **没有输出**：继续往下做。

```bash
fallocate -l 1G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
swapon --show
```

最后一行能看到 `/swapfile` 就成功了。OpenVZ 类机器可能不允许自建 swap，个别文件系统上 `fallocate` 也会失败，报错就跳过。

> 这段只执行一次。重复执行会在 `/etc/fstab` 里写出重复行。

### 7.5 开 BBR（可选）

BBR 是 Google 出的拥塞控制算法，跨境线路上常能明显改善速度。

**误判坑**：直接查「可用算法」只会列出**已加载**的模块。很多机器上 bbr 明明能用，只是没加载，列表里只有 `reno cubic`。先尝试加载，再查：

```bash
modprobe tcp_bbr
sysctl net.ipv4.tcp_available_congestion_control
```

`modprobe` 报错的（常见于 OpenVZ / LXC），说明这台机器用不了，跳过。列表里有 `bbr` 的，写进独立文件启用（用独立文件，重复执行也不会写出重复行）：

```bash
printf 'net.core.default_qdisc=fq\nnet.ipv4.tcp_congestion_control=bbr\n' > /etc/sysctl.d/99-bbr.conf
sysctl --system
sysctl net.ipv4.tcp_congestion_control
```

显示 `bbr` 就是开上了。

### 7.6 时区和主机名（可选）

```bash
timedatectl set-timezone Asia/Shanghai
hostnamectl set-hostname 给机器起个名字
```

- 时区对上自己的作息，看日志省事。机器多、跨时区的，统一用 UTC 也是个好选择。
- 主机名改完，同步改 `/etc/hosts` 里 `127.0.1.1` 那一行，不然 `sudo` 时可能报 `unable to resolve host`。
- 云镜像的 cloud-init 可能在重启后把主机名改回去。遇到的话，在 `/etc/cloud/cloud.cfg` 里把 `preserve_hostname` 设为 `true`。

---

## 第 8 步：最后锁门——SSH 加固

> [!CAUTION]
> 深呼吸。这一步是全篇**唯一**能把自己锁在门外的操作。**旧窗口保持开着**，直到 8.3 的验证全部通过。

### 8.1 把加固写进一个独立的 drop-in 文件

**不要直接改主配置文件 `sshd_config`。**

原因：Ubuntu 和很多云镜像里，`sshd_config.d/` 子目录下还有个 `50-cloud-init.conf`，里面常写着 `PasswordAuthentication yes`。而 sshd 的规矩是：**同一个选项，先读到的那一行生效。**你在主文件里改成 `no`，很可能根本没生效，却以为密码登录已经关了。

稳妥做法：新建一个排在最前面的文件，让你的配置先被读到。

**先确认主配置会加载这个目录**（Debian 11+ / Ubuntu 20.04+ 默认都有）：

```bash
grep -i '^Include' /etc/ssh/sshd_config
```

应该看到 `Include /etc/ssh/sshd_config.d/*.conf`，而且它在文件**开头**附近。没有这一行的老系统，drop-in 目录根本不会被读取，写了等于白写。这种情况就在 `sshd_config` 的**第一行**手动加上这句 Include。

然后写加固文件：

```bash
cd /etc/ssh
cat > sshd_config.d/00-hardening.conf <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
```

| 配置 | 作用 |
|---|---|
| `PermitRootLogin no` | root 禁止直接从门口进，先用普通用户登录再 sudo |
| `PasswordAuthentication no` | 彻底关掉密码登录，只认钥匙 |
| `KbdInteractiveAuthentication no` | 关掉键盘交互认证。**别漏**，某些 PAM 配置下它仍然能输密码，只关第二行等于留了扇后门 |

### 8.2 让配置生效，再验真

```bash
sshd -t
sshd -T | grep -Ei '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication)'
systemctl restart ssh
```

1. `sshd -t` 查语法，**没输出就是没毛病**。
2. `sshd -T` 看实际生效值。要看到下面三个都对才算数：

   ```
   permitrootlogin no
   passwordauthentication no
   kbdinteractiveauthentication no
   ```

   对不上，说明还有别的 drop-in 在顶你的配置。去那个文件把同一项改掉，再查一次。
3. `systemctl restart ssh` 让新配置生效。**这一步不能省**，漏了就是自以为锁了门，其实门一直开着。

> [!NOTE]
> 新版 Ubuntu 用 `ssh.socket` 监听，`sshd -T` 显示的 port 是配置文件里的值，不一定是真实监听端口。真实监听以第 9 步的 `ss -tlnp` 为准。

### 8.3 验证：一正一反，两个都要过

新开窗口，**正向验证**（应该进得去）：

```bash
ssh 用户名@你的IP
```

不用输账号密码直接进，对。（私钥设了口令的，这里输的是钥匙口令；agent 已经解锁过的，什么都不用输。）

**反向验证**（应该进不去才对）：强制不用钥匙，只用密码试一次：

```bash
ssh -o PubkeyAuthentication=no 用户名@你的IP
```

预期结果是 `Permission denied`。**这条才是密码登录真的关了的证据**；「用钥匙能进」只能证明钥匙能用，证明不了密码已关。

再顺手试 `ssh root@你的IP`，同样应该被拒。

**两个方向都对了，才回来关掉旧窗口。**

### 8.4 改端口（可选，别勉强）

先说清定位：**改端口不算真正的安全加固。**它的作用只是躲开冲着 22 来的自动化扫描、让日志清净点，代价是安全组、防火墙、fail2ban、监听方式四处都要联动。「22 端口 + 纯密钥 + 禁 root」已经是完整防线，怕麻烦完全可以不改。

要改，按这个顺序（错一步就锁门）：

**① 安全组放行新端口**

去商家面板的安全组放行新端口。端口选 10000–30000 之间，比如 `22222`。别选 `52222` 这种落在系统临时端口范围（32768–60999）里的，极小概率会和出站连接撞车。

**② 防火墙放行新端口**（旧端口规则先留着）

```bash
ufw allow 22222/tcp
```

**③ 写进加固文件**

在 `00-hardening.conf` 里加一行 `Port 22222`。

注意 `Port` 和其他选项不同，**它是多值叠加的**：主文件里如果还有一行没注释的 `Port 22`，两个端口会同时监听。把主文件和 drop-in 目录里其他 `Port` 行都注释掉，再确认只剩一行：

```bash
sshd -T | grep ^port
```

**④ 按系统类型重启监听**

先判断你这台有没有 `ssh.socket`：

```bash
systemctl is-active ssh.socket
```

<table>
<tr><th>输出</th><th>怎么做</th></tr>
<tr>
<td><code>inactive</code><br>或找不到这个单元<br>（Debian、Ubuntu 22.04 及更早）</td>
<td>

监听不归 `ssh.socket` 管，别去动它，只需要：

```bash
systemctl restart ssh
```

</td>
</tr>
<tr>
<td><code>active</code><br>（Ubuntu 22.10 及以后）</td>
<td>

**Ubuntu 24.04 及以后**：系统会自动从 sshd 配置里读取 `Port`，重新加载即可：

```bash
systemctl daemon-reload
systemctl restart ssh.socket
```

**Ubuntu 22.10 / 23.04**：只改配置文件不生效，要手动改 socket：

```bash
systemctl edit ssh.socket
```

在编辑区写入（第一行 `ListenStream=` 空着是清空旧监听，必须写）：

```ini
[Socket]
ListenStream=
ListenStream=22222
```

然后：

```bash
systemctl daemon-reload
systemctl restart ssh.socket
systemctl restart ssh
```

> 不确定版本的，用 22.10 的手动方法，24.04 上也同样有效。

</td>
</tr>
</table>

**⑤ 确认新端口在监听，新窗口用新端口登录**

```bash
ss -tlnp | grep 22222
ssh -p 22222 用户名@你的IP
```

**⑥ 都通了，才收尾**

- `ufw delete allow 22/tcp` 删掉旧端口的防火墙规则；
- 安全组里的旧端口规则一并回收；
- 更新自己电脑 `~/.ssh/config` 里的 `Port`，以后还是 `ssh myvps` 一把梭。

### 8.5 fail2ban 收尾（改了端口必做，不改也建议做）

把 fail2ban 的端口和 SSH 对齐，并给自己留一条白名单。用下面的内容**覆盖** 7.2 写的 `/etc/fail2ban/jail.local`：

```bash
cat > /etc/fail2ban/jail.local <<'EOF'
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled = true
backend = systemd
port = 22222
EOF
```

- **没改端口的**，删掉 `port = 22222` 这一行，用默认值即可。
- `backend = systemd` 要保留，原因见 7.2。
- `ignoreip` 里可以加上你自己的**固定 IP**（空格分隔），免得手滑输错几次被自己的保安关在门外。没有固定 IP 就只留这两个本机地址，别乱填。

```bash
systemctl restart fail2ban
sleep 2
fail2ban-client status sshd
```

jail 正常，保安正式上岗。

---

## 第 9 步：重启总验收（最重要的一步）

```bash
reboot
```

重启后，**只用「新用户 + 密钥 +（新端口）」登录**，逐项检查。

> [!WARNING]
> **含 `sudo` 的多行命令不要整段粘贴。**第一条 `sudo` 停下来等密码时，后面粘贴进去的几行会被当成「密码」吞掉，表现为连续几次 `Sorry, try again`，后面的命令都没执行。
>
> 正确做法：先单独执行 `sudo -i` 输入密码切到 root，再粘贴不带 `sudo` 的命令；或者先单独执行一次 `sudo -v` 缓存密码。

先单独执行：

```bash
sudo -i
```

提示符变成 `root@...#` 后，整段粘贴，一次跑完所有检查：

```bash
echo "== sshd ==";        sshd -T | grep -Ei '^(port|permitrootlogin|passwordauthentication|kbdinteractiveauthentication)'
echo "== ufw ==";         ufw status
echo "== 监听端口 ==";     ss -tlnp
echo "== fail2ban ==";    fail2ban-client status sshd | head -4
echo "== swap ==";        swapon --show
echo "== BBR ==";         sysctl net.ipv4.tcp_congestion_control
echo "== 主机名/时区 ==";  hostname; timedatectl | grep 'Time zone'
echo "== 失败服务 ==";     systemctl --failed --no-legend
echo "== 内核 ==";        uname -r
```

> `ss -tlnp` 里看到 `127.0.0.53` / `127.0.0.54` 的 53 端口是 Ubuntu 的本机 DNS（systemd-resolved），只监听本机地址，外部访问不到，可以忽略。`fail2ban` 的 `Total failed` 有几十次也正常，那是公网扫描器在试，全被拒了。

对照下面的清单逐项打勾。

**正向（应该成立）：**

- [ ] 能登录，`sudo` 能提权
- [ ] `sshd -T` 四个值都是你设的样子
- [ ] `ufw status` 规则还在，只有该开的端口
- [ ] `ss -tlnp` 所有监听端口都是你预期的那几个。**这是安全基线最直观的一张体检表**
- [ ] `fail2ban-client status sshd` 正常工作
- [ ] `free -h` 能看到 swap（如果加了）
- [ ] `sysctl net.ipv4.tcp_congestion_control` 重启后仍是 `bbr`（如果开了）
- [ ] `hostname`、`timedatectl` 主机名和时区没被打回去
- [ ] `systemctl --failed` 没有报错的服务

**反向（应该被拒，成立才算加固是真的）：**

- [ ] `ssh root@你的IP` 被拒
- [ ] `ssh -o PubkeyAuthentication=no 用户名@你的IP` 被拒

**为什么非要重启验？**很多设置「当时生效、重启失效」，swap、BBR、主机名都是典型。不重启验一次，你永远不知道哪颗雷埋着。

> [!TIP]
> 验收不过别慌。第 0 步认清的救命门（VNC，密码在第 2、5 步都实测过手敲能进）和商家面板的重置 / 救援功能，就是为这一刻准备的。

---

## 第 10 步：验收通过，再打快照、确认续费

**快照一定要等验收过了再打。**验收之前打的「出厂状态」，存的可能是一个还没验证、甚至有问题的系统，还原回去等于还原一个坑。

- **快照 ≠ 备份。**快照通常存在同一家商家那里，商家出事它跟着没。重要资料仍要另存一份到别处。
- 确认快照收不收费，别被默默扣钱。
- **看清续费价格和到期时间，设一个提前提醒。**忘了续费被回收，是新手最冤的一种死法。

---

## 完工清单

复制到你自己的笔记里，对着打勾：

**打底（第 0～6 步）**

- [ ] 初始密码已改，VNC 实测（root 或默认用户）手敲能登录
- [ ] 系统已更新，重启需求已处理
- [ ] 配置、IP 归属、虚拟化类型已核对
- [ ] 日常普通用户已建（或默认用户已设密码），sudo 可用，VNC 实测能手敲
- [ ] 私钥已设口令、已备份；预置公钥已检查
- [ ] 密钥登录已测通

**安防与锁门（第 7～8 步）**

- [ ] 防火墙已开，当前 SSH 端口已放行，只留必要端口（v6 一并管住）
- [ ] fail2ban 已确认在工作（`backend = systemd`），端口与 SSH 对齐，白名单已配
- [ ] 自动安全更新已开
- [ ] 主配置确认有 `Include sshd_config.d`
- [ ] `sshd -T` 核对：密码登录关、键盘交互关、root 直登关
- [ ] 改了端口的：安全组已放行、新端口监听已确认、旧端口规则已删

**验收与存档（第 9～10 步）**

- [ ] `ss -tlnp` 自查：无意外监听端口
- [ ] 反向测试通过：root 登录被拒、强制密码登录被拒
- [ ] 重启验收全部通过（含 BBR / 主机名 / 时区不回滚）
- [ ] 验收后快照已打；续费时间和提醒已确认

全勾完，这台机器才真正归你。

---

## 装应用之前

安全基线打好了，接下来往里装什么、用它干什么，就看你自己的需求了。

> [!WARNING]
> **如果你要装 Docker，先记住一句：Docker 发布的端口会绕过 ufw。**你以为被防火墙挡在外面的端口，可能正对全网开着。别带着「防火墙已经管住一切」的印象去部署服务，发布端口后用 `ss -tlnp` 和外部端口扫描再确认一遍。

发现步骤有误、或者在某个系统版本上跑不通，欢迎提 [Issue](https://github.com/kungen1008/vps-setup/issues) 告诉我。觉得有用的话，点个 ⭐ Star。

---

## 附：英文名词小词典

<details>
<summary><b>点开查看（按文中出现顺序）</b></summary>

| 名词 | 解释 |
|---|---|
| **VPS**（Virtual Private Server，虚拟专用服务器） | 你在海外租的那间「仓库」，一台 24 小时开机的远程机器 |
| **IP / IPv6**（Internet Protocol，网际协议） | 机器在网络上的「门牌号」。IPv6 是新一代、更长的门牌号，和 IPv4 并行存在 |
| **SSH**（Secure Shell，安全外壳） | 远程登录服务器的加密通道，默认走 22 号端口 |
| **端口 / TCP** | 端口是门牌号下的「门洞编号」。TCP 是最常用的传输方式，`22/tcp` 就是「22 号门洞的 TCP 通道」 |
| **root** | 系统里权力最大的账号，包括一条命令把整台机器删光 |
| **sudo**（superuser do） | 普通用户临时借用管理员权力的方式，每次借都要输密码 |
| **VNC**（Virtual Network Computing） | 商家面板里的网页版「显示器」，不走 SSH，被锁在门外时的救命门 |
| **apt**（Advanced Package Tool） | Debian / Ubuntu 的「软件管家」，装软件、更新系统都找它 |
| **CPU** | 机器的「大脑」，`nproc` 数的就是它的核数 |
| **kvm / openvz / lxc** | 虚拟化类型。KVM 是独享内核，最正经；OpenVZ 和 LXC 和邻居共享内核，部分功能用不了 |
| **公钥 / 私钥** | 一对钥匙。私钥留在你电脑里，公钥挂到服务器上，登录时两边对暗号 |
| **ed25519** | 目前主流的密钥类型，又短又快又安全 |
| **passphrase**（口令） | 给私钥再加的一把小锁，私钥文件被偷也用不了 |
| **ssh-agent** | 帮你「记住已解锁私钥」的小程序，输一次口令后不再反复输 |
| **authorized_keys** | 服务器上「允许进门的公钥」名单文件，一行一把钥匙 |
| **ufw**（Uncomplicated Firewall） | Ubuntu 自带的简易防火墙工具 |
| **fail2ban / jail** | 盯着日志的保安，发现某个 IP 反复试密码就临时拉黑。jail 是它的一条监控规则 |
| **sshd**（SSH daemon） | 服务器上接待 SSH 连接的后台程序 |
| **systemd / systemctl / journald** | 管理后台服务的系统组件和命令；journald 是它自带的日志系统 |
| **drop-in 文件** | 把某项配置拆出来、单独放进专门目录的小文件，主程序按文件名顺序读取 |
| **PAM**（Pluggable Authentication Modules） | Linux 里管「怎么验证你就是你」的认证模块 |
| **cloud-init** | 云镜像开机时自动做初始设置的程序，比如设主机名、预置密钥 |
| **swap**（交换空间） | 内存不够时拿一块硬盘空间顶上，免得直接死机 |
| **BBR** | Google 做的网络拥塞控制算法，跨境线路上常能明显提速 |
| **snapshot**（快照） | 给整台机器某一刻的状态拍张照，出事能还原。不等于备份 |
| **UTC** | 世界标准时间，跨时区对日志时用它统一 |
| **Docker** | 把应用打包成「集装箱」运行的工具 |
| **RHEL / SELinux / dnf / firewalld** | 红帽系 Linux 及其安全模块、包管理器、防火墙服务，和 Debian / Ubuntu 是两套体系 |

</details>
