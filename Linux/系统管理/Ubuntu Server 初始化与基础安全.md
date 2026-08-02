---
title: Ubuntu Server 初始化与基础安全
aliases:
  - Ubuntu Server 开发机初始化与安全基线
  - Ubuntu Server 初始化
  - Ubuntu 开发机安全基线
  - Linux 开发机首次配置
tags:
  - Linux
  - Linux/系统管理
  - Linux/安全
  - Ubuntu
created: 2026-07-16T00:31:57
updated: 2026-08-02T20:54:15
---

本文给出一台新装 Ubuntu Server 的通用初始化顺序：先保留控制台恢复入口，再核对身份、主机名、时区、网络、DNS、时间和软件包，最后核对或建立 OpenSSH 入口、UFW 与服务基线。

这里的目标是得到一台可维护的学习或开发主机，不是直接套用生产服务器的加固清单。命令行的通用阅读方法见 [[Linux 命令行学习路线与命令地图]]，用户与权限详见 [[Linux 用户、用户组、sudo 与文件权限]]，软件包管理详见 [[APT 软件包管理基础]]，服务和日志详见 [[systemd 服务与日志基础]]，网络字段与验证命令详见 [[Linux 网络接口、IP 地址、路由与 DNS 基础]]，远程登录详见 [[OpenSSH 密钥登录、服务端配置与排查]]，主机防火墙原理与规则管理详见 [[Linux 主机防火墙与 UFW 基础]]。

> [!info] 核对日期与适用范围
> 本文于 **2026-07-20** 核对 Ubuntu Server、UFW 和 systemd 官方资料。不同 Ubuntu 版本、镜像预装组件和网络环境可能不同，执行时应以 `/etc/os-release`、本机手册和当前官方文档为准。

## 本篇掌握目标

- **必须熟练**：执行前分清只读检查与系统变更，理解 `sudo` 只提升其后单条命令的权限；能拆出变量、引用、管道、重定向、条件和退出状态，并根据输出验证结果。
- **理解会查**：知道 `uname`、`id`、`hostnamectl`、`timedatectl`、`lsblk`、`df`、`ip`、`systemctl`、`journalctl`、`apt` 和 `ss` 分别回答什么问题，具体字段和低频参数可随时查手册。
- **认识即可**：复杂 `grep`、`dpkg-query`、`systemctl show` 选项和发行版差异；用到时回到对应专题或当前系统手册核对。

## 完成标准

- 普通用户能够登录，并已实际验证需要的 `sudo` 权限。
- 主机名与时区符合用途，系统时间已同步。
- 主机拥有预期地址和默认路由，DNS 与 HTTPS 访问正常。
- APT 索引可更新，待升级项目已经过审查。
- `systemctl --failed` 中没有未解释的失败单元。
- OpenSSH 已按需安装并核对激活路径，UFW 启用前已经保留控制台和可验证的 SSH 入口。
- 已把初始化后的系统状态保存到受保护的本地基线文件，并确认文件完整、权限正确且没有主动采集凭据或完整日志。

## 1. 先保留恢复入口

初始化期间不要同时修改网络、SSH 和防火墙。推荐顺序如下：

```mermaid
flowchart TD
    A["保留虚拟机或物理机控制台"] --> B["只读盘点"]
    B --> C["验证普通用户与 sudo"]
    C --> D["设置主机名与时区"]
    D --> E["验证网络、DNS 与时间"]
    E --> F["更新软件包"]
    F --> G["建立并验证 OpenSSH 会话"]
    G --> H["保留已验证会话，先放行 SSH，再启用 UFW"]
    H --> I["启用后新建 SSH 会话复测"]
```

> [!warning] 控制台是网络配置失败时的恢复入口
> 在 UFW 启用后的新 SSH 会话通过验证前，不要关闭控制台和原有可用会话。远程修改网络、`sshd` 或 UFW 后立刻断开唯一连接，可能把自己锁在主机外。

## 2. 只读盘点当前系统

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
printf '%s\n' '--- system ---'
# 显示系统发行版、版本和标识信息。
cat /etc/os-release
# 显示内核、主机和硬件架构等完整系统信息。
uname -a
# 显示 Debian 软件包体系使用的主 CPU 架构。
dpkg --print-architecture

printf '%s\n' '--- identity ---'
# 显示当前 Shell 所属的用户名。
whoami
# 显示当前用户的 UID、GID 及附属组编号。
id
# 显示当前用户所属的用户组名称。
groups
printf 'HOME=%s\n' "$HOME"

printf '%s\n' '--- hostname and time ---'
# 显示主机名及其静态、瞬态和美化名称等信息。
hostnamectl
# 显示本地时间、时区和时间同步状态。
timedatectl status

printf '%s\n' '--- storage ---'
# 按指定列列出块设备、文件系统类型和挂载点。
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
# 以易读单位显示已挂载文件系统的类型、容量和使用情况。
df -hT

printf '%s\n' '--- network ---'
# 以精简格式显示各网络接口的状态和已配置地址。
ip -brief address
# 显示内核路由表，重点核对默认路由。
ip route
# 显示 systemd-resolved 的 DNS 配置和当前解析状态。
resolvectl status

printf '%s\n' '--- failed units ---'
# 列出状态为失败的 systemd 单元，不使用分页器。
systemctl --failed --no-pager
```

重点确认：

- 当前用户不是 root，家目录与预期一致。
- 系统版本和 CPU 架构符合将要安装的软件。
- 根文件系统空间充足。
- 存在非回环地址和默认路由。
- 失败单元能够逐一解释；不要看到失败就直接禁用服务。

## 3. 验证普通用户与 sudo

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
# 刷新 sudo 的认证缓存；必要时会提示输入当前用户密码。
sudo -v
# 列出当前用户可通过 sudo 执行的命令及其限制。
sudo -l
# 以 root 身份执行 id，确认 sudo 提权是否生效。
sudo id
```

`sudo id` 预期显示 `uid=0(root)`，命令结束后仍回到普通用户 Shell。`sudo` 是临时提升某条命令的权限，不应把 root 直接登录当作日常工作方式。

用户、组、目录权限、`umask`、新增账号和安全修复方式见 [[Linux 用户、用户组、sudo 与文件权限]]。如果当前用户没有管理权限，应使用安装时创建的管理员账号或控制台恢复，不要复制来源不明的 sudoers 配置。

## 4. 设置主机名

主机名用于 Shell 提示符、日志和远程识别。建议使用小写字母、数字和连字符，避免空格、真实姓名、设备序列号和可能变化的 IP。

先记录旧值，再输入新值：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
(
old_hostname="$(hostnamectl --static)"
printf '当前主机名：%s\n' "$old_hostname"
printf '请输入新的主机名；不修改则按 Ctrl-C：'
IFS= read -r new_hostname

case "$new_hostname" in
  ''|*[!a-z0-9-]*|-*|*-) printf '主机名格式不符合本文约束。\n' >&2; exit 1 ;;
esac

sudo hostnamectl set-hostname "$new_hostname"
hostnamectl status
)
```

随后检查 `/etc/hosts`：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
# `-n` 输出匹配行的行号；`-E` 启用扩展正则，以便匹配以本机回环地址开头的 hosts 条目。
grep -nE '^(127\.0\.0\.1|127\.0\.1\.1|::1)[[:space:]]' /etc/hosts
# 用当前静态主机名查询系统名称解析结果（如 `/etc/hosts`）；未解析到时由 `|| true` 忽略非零退出码。
getent hosts "$(hostnamectl --static)" || true
```

Ubuntu 常用 `127.0.1.1` 映射本机主机名。如果该行仍是旧名称，先备份，再用 `sudoedit` 精确修改：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
(
if ! hosts_backup="$(sudo mktemp /etc/hosts.before-hostname.XXXXXX)"; then
  printf '%s\n' '停止：无法创建 /etc/hosts 备份路径。' >&2
  exit 1
fi
if sudo cp -a -- /etc/hosts "$hosts_backup"; then
  printf 'hosts_backup=%s\n' "$hosts_backup"
  sudoedit /etc/hosts
else
  sudo rm -f -- "$hosts_backup"
  printf '%s\n' '停止：/etc/hosts 备份失败，未打开编辑器。' >&2
  exit 1
fi
)
```

不要用不受约束的全局替换修改 `/etc/hosts`。如修改后 `sudo` 提示无法解析本机名，可从控制台恢复刚创建的备份，或修正 `127.0.1.1` 对应项。

## 5. 设置时区并验证时间同步

列出可用时区，输入目标值：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
(
timedatectl list-timezones
printf '请输入 timedatectl 列表中的时区：'
IFS= read -r target_timezone

# `-q` 静默仅用退出码判断，`-x` 要求整行完全匹配，`-F` 将输入按普通文本而非正则表达式处理。
if ! timedatectl list-timezones | grep -qxF "$target_timezone"; then
  printf '停止：不是 timedatectl 列出的时区：%s\n' "$target_timezone" >&2
  exit 1
fi

sudo timedatectl set-timezone "$target_timezone"
timedatectl status
)
```

以上两个输入块使用圆括号创建子 Shell；格式校验中的 `exit` 只停止当前代码块，不会退出控制台或 SSH 登录 Shell。

预期 `Time zone` 为所选值，`System clock synchronized` 最终为 `yes`。查看同步服务与日志：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
# 查看 systemd-timesyncd 服务的当前状态；`--no-pager` 直接输出，不进入交互式分页器。
systemctl status systemd-timesyncd.service --no-pager
# 查看该服务自本次启动（`-b`）以来的最近 80 条日志；`-u` 按服务筛选，`--no-pager` 禁用分页。
journalctl -u systemd-timesyncd.service -b --no-pager -n 80
```

如果镜像使用其他 NTP 客户端，应以实际启用的服务为准，不要同时运行多个未经协调的时间同步服务。

## 6. 分层验证网络、DNS 与 HTTPS

下列命令的字段含义与判断方法见 [[Linux 网络接口、IP 地址、路由与 DNS 基础]]。

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
# 以精简格式查看各网络接口的状态、MAC 地址和已配置的 IP 地址。
ip -brief address
# 查看内核路由表，重点确认是否存在指向网关的默认路由。
ip route
# 查看 systemd-resolved 的全局及各接口 DNS 配置与当前解析状态。
resolvectl status
# 通过 NSS 的 `ahosts` 查询域名可用于连接的地址；不同于上文的 `hosts`（查询 hosts 数据库条目以核对本机主机名映射），`ahosts` 基于 `getaddrinfo()` 返回地址，同一 IP 可能因套接字类型重复显示。
getent ahosts archive.ubuntu.com
# `-I` 仅请求 HTTP 响应头以验证 HTTPS 连通性；`--max-time 15` 将 DNS、连接和传输在内的总耗时限制为 15 秒。
curl -I --max-time 15 https://archive.ubuntu.com/
```

判断顺序：

1. 网卡是否有预期地址。
2. 是否存在默认路由。
3. DNS 是否能解析官方域名。
4. HTTPS 是否能完成连接和证书校验。

`curl -I` 返回 HTTP 响应即说明网络路径基本成立，不要求状态码一定是 `200`。如果 DNS 失败，不要通过关闭 TLS 校验或随意替换来源不明的软件源来掩盖问题。

Ubuntu Server 常由 Netplan 管理持久网络配置。修改 `/etc/netplan/*.yaml` 前必须保留控制台；远程环境优先使用带自动回滚窗口的 `sudo netplan try`，确认连通后再接受配置。

## 7. 更新 APT 软件包

APT 的软件源、索引、已安装状态和升级行为见 [[APT 软件包管理基础]]。本节只保留初始化主线需要的最小操作。

先更新索引并审查升级内容：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
sudo apt update
apt list --upgradable
```

确认来源、磁盘空间和变更范围后再升级：

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
sudo apt upgrade
```

`apt upgrade` 可能需要交互确认。内核、OpenSSH、网络组件或需要重启的更新，应在仍有控制台恢复路径时进行。下面使用 `test -f` 判断重启标记是否为普通文件，条件命令见 [[Shell 脚本阅读基础#6. 使用 test 表达条件|test 条件判断]]。

**执行位置：Ubuntu Server（控制台，任意目录）**

```bash
# Ubuntu 在需要重启以完成更新时创建此标记文件；`-f` 同时确认它存在且是普通文件。
if test -f /var/run/reboot-required; then
  # 输出系统记录的重启提示。
  cat /var/run/reboot-required
  # `.pkgs` 会列出触发该提示的软件包；文件可能不存在，故隐藏错误并以成功状态继续。
  cat /var/run/reboot-required.pkgs 2>/dev/null || true
else
  # 没有标记文件通常表示当前更新未要求重启。
  printf '%s\n' '当前未检测到 reboot-required 标记。'
fi
```

如果 `apt update` 或 `apt upgrade` 提示无法获取锁，按 [[APT 软件包管理基础#5.5 处理锁冲突与重启提示]] 区分正常占用、异常中断和重启提示。

## 8. 建立并验证 OpenSSH 会话

如果已在 Ubuntu Server 安装器的 **SSH Setup** 页面勾选 **Install OpenSSH server**，安装器已经安装 `openssh-server`，无需重复安装；如果没有勾选，先按 [[APT 软件包管理基础]] 安装该软件包。

随后从 [[OpenSSH 密钥登录、服务端配置与排查#2. 在服务端准备 sshd]] 开始完成服务端检查与客户端登录。socket activation、有效端口、主机指纹、用户密钥和认证收紧均由该专题说明，本初始化清单只保留进入 UFW 前必须达到的状态。

> [!success] 进入 UFW 步骤前的返回检查点
> - Ubuntu Server 控制台仍可登录。
> - OpenSSH 的实际激活路径、有效配置和监听端口已经核对。
> - 客户端已核对主机指纹并成功登录；保留一条可用会话，同时从另一终端重新登录成功。
>
> 如果还没有成功建立客户端 SSH 会话，不进入第 9 节。

## 9. 安全启用 UFW

本节只负责把 Ubuntu 初始化主线交给 UFW 专题，不再维护另一套容易漂移的防火墙命令。继续前，第 8 节保留的控制台和可用 SSH 会话必须仍可使用，并且已经从另一终端完成一次启用前重新登录。

如果尚未理解主机防火墙的连接层次、规则要素、UFW 状态和验证边界，先学习 [[Linux 主机防火墙与 UFW 基础]] 第 1～5 节；随后严格按照 [[Linux 主机防火墙与 UFW 基础#6. 实践：在保留 SSH 管理入口的前提下首次启用 UFW]] 完成当前主机操作。是否可以套用首次启用流程、何时必须停止，以及 OpenSSH application profile（应用配置档案）与实际端口不一致时如何选择规则，均以该专题的前置条件和分支为准。

> [!success] 进入第 10 节前的返回检查点
> - UFW 的运行状态为 `active`，默认策略与已保存用户规则均已按当前主机需求核对。
> - SSH 的有效配置、真实监听端口与 UFW 允许规则一致。
> - 启用 UFW 后已经从真实客户端建立并稳定使用一条全新的 SSH 会话。
> - 在新会话验证成功前，控制台和启用前保留的基准会话始终保持可用。

如果启用后的新会话失败，不要关闭控制台或基准会话，也不要在保存现场前直接停用或重置 UFW。先按 [[Linux 主机防火墙与 UFW 基础#7.2 新连接失败时保留现场并恢复入口]] 保留证据并判断是否需要临时恢复入口，再按 [[Linux 主机防火墙与 UFW 基础#7.3 按层次定位]] 查明原因。完成修正、重新启用和新会话验证前，不进入第 10 节。

## 10. 检查服务与日志基线

第 2 节只记录初始化前的失败 unit。本节在软件升级、OpenSSH 和 UFW 配置完成后重新检查，确认这些操作没有留下无法解释的系统异常。

这里需要依次回答三个不同问题：

1. **当前失败视图**：是否有 unit 处于 `failed` 状态。
2. **启用配置视图**：哪些已安装 unit 文件被配置为在开机或其他触发条件下参与激活；这不等于它们当前正在运行。
3. **当前启动日志视图**：本次启动中有哪些 warning 及更严重级别的消息；有日志不等于一定存在故障，需要结合来源、时间和实际影响判断。

先按 [[systemd 服务与日志基础#9. 综合检查：失败状态、启用关系与本次启动日志]] 完成三层检查。不要因为名称陌生就直接 `disable`、`stop`、`reset-failed` 或删除 unit，也不要为了得到空日志而忽略警告。

> [!success] 进入第 11 节前的返回检查点
> - 没有未解释的 failed unit；存在失败项时，已经通过目标 unit 的状态与日志定位原因。
> - 能区分启用状态与当前运行状态，且没有发现无法解释的非预期启用项。
> - 当前启动中的重要警告已经结合来源和实际影响审查；不要求日志必须为空。

## 11. 保存系统状态基线

第 10 节是在终端中观察当前状态；本节把其中适合长期比较的结果连同系统、时间、网络、OpenSSH 和 UFW 摘要保存为一个带时间和用途的本地文本文件。该文件用于回答“之后发生了什么变化”，不能恢复系统，也不能单独证明主机健康或安全。

按照 [[Ubuntu Server 状态基线的采集与比较#4. 采集一份完整的初始化后基线]] 执行完整示例，本次用途输入 `post-initialization`。专题笔记会逐段解释文件创建、采集内容、失败边界、权限检查和后续差异比较，本初始化主线不再复制同一套脚本。

基线不会主动采集密码、令牌、私钥、`/etc/shadow`、完整环境变量或完整 journal，但仍包含用户名、主机名、地址、路由和防火墙规则等主机信息。应只保存在受保护的位置；未经审查和脱敏，不要提交到 Git、同步到公开位置或直接发给他人。

> [!success] 初始化完成前的返回检查点
> - 文件末尾存在 `status=complete`，不是采集中断后留下的部分文件。
> - 文件和目录权限已经核对，权限位没有向同组或其他普通用户授予读取权。
> - 已人工检查采集范围，没有加入凭据、完整日志或其他不应保存的内容。
> - 已记录文件路径、采集时间和用途；知道它是状态记录而不是系统备份。

## 常见问题

### 主机能获得地址，但不能解析域名

按 `ip route`、`resolvectl status`、`getent ahosts` 的顺序排查。先区分路由与 DNS，再修改配置。

### 时间一直不同步

确认网络和 DNS，检查实际 NTP 服务日志，并确认没有多个时间服务争用。虚拟机暂停恢复后可能需要短暂重新校时。

### `apt` 提示无法获取锁

不要直接删除 `/var/lib/dpkg/lock*`；锁文件消失并不等于占用任务安全结束。检查命令、判断顺序和异常中断后的恢复方式统一见 [[APT 软件包管理基础#5.5 处理锁冲突与重启提示]]。

### `sshd -t` 提示 `Missing privilege separation directory: /run/sshd`

这通常需要先区分“配置语法错误”和“绕过 systemd 后运行时目录尚未创建”。按 [[OpenSSH 常用命令基础#5.4 sshd -t 提示缺少 /run/sshd]] 核对单元机制、启动前检查和服务日志，不要把手工创建目录当作持久修复。

### UFW 启用后 SSH 超时

保留控制台和仍可用的基准会话，先按 [[Linux 主机防火墙与 UFW 基础#7.2 新连接失败时保留现场并恢复入口]] 保存 UFW 状态、规则与真实监听信息，再按 [[Linux 主机防火墙与 UFW 基础#7.3 按层次定位]] 分层排查。只有确实需要先恢复新的远程管理入口时，才在保留现场后从控制台临时停用 UFW。

### `systemctl --failed` 出现服务

先运行 `systemctl status` 和 `journalctl -u` 理解失败原因。不要为了得到空列表就盲目 `disable` 或删除单元。

## 初始化检查清单

- [ ] 控制台恢复入口在整个初始化期间保持可用。
- [ ] 普通用户与 `sudo` 权限已经实际验证。
- [ ] 主机名、`/etc/hosts` 和时区符合预期。
- [ ] 地址、默认路由、DNS、HTTPS 和时间同步均正常。
- [ ] APT 索引已更新，升级内容经过审查。
- [ ] OpenSSH 语法、激活单元和监听状态已验证。
- [ ] UFW 先放行管理入口，再启用并用新会话复测。
- [ ] 失败服务与重要警告日志均已解释。
- [ ] 系统状态基线已完成采集，文件完整性、权限和内容边界已经检查。

## 官方参考资料

- [Ubuntu Server：用户管理](https://documentation.ubuntu.com/server/how-to/security/user-management/)
- [Ubuntu Server：软件包管理](https://documentation.ubuntu.com/server/how-to/software/package-management/)
- [Ubuntu Server：网络配置](https://documentation.ubuntu.com/server/explanation/networking/configuring-networks/)
- [Ubuntu Server：使用 timedatectl 与 timesyncd](https://documentation.ubuntu.com/server/how-to/networking/timedatectl-and-timesyncd/)
- [Ubuntu Server：OpenSSH Server](https://documentation.ubuntu.com/server/how-to/security/openssh-server/)
- [Ubuntu 24.04 LTS 发布说明：OpenSSH 的 systemd socket activation](https://documentation.ubuntu.com/release-notes/24.04/#openssh)
- [Ubuntu Server：UFW 防火墙](https://ubuntu.com/server/docs/how-to/security/firewalls/)
- [Netplan：安全应用配置](https://netplan.readthedocs.io/en/stable/netplan-try/)
- [systemd：hostnamectl](https://www.freedesktop.org/software/systemd/man/latest/hostnamectl.html)
- [systemd：timedatectl](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
