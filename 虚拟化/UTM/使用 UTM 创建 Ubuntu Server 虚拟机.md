---
title: 使用 UTM 创建 Ubuntu Server 虚拟机
aliases:
  - 使用 UTM 创建 Ubuntu Server 开发虚拟机
  - UTM 安装 Ubuntu Server
  - Ubuntu Server 虚拟机搭建
tags:
  - 虚拟化
  - UTM
  - Ubuntu
  - Linux/安装
created: 2026-07-16T00:31:57
updated: 2026-07-17T01:12:07
---

本文是一篇可复用的 UTM 操作笔记：从选择虚拟化方式、校验 Ubuntu Server ISO 开始，完成虚拟机创建、系统安装、安装介质卸载和首次启动验证。它不规定唯一的 CPU、内存、磁盘或 Ubuntu 版本；具体选择应先参考 [[虚拟机、客户机与 CPU 架构]] 和 [[UTM 虚拟机网络与资源规划]]。

> [!info] 资料核对日期
> 本文涉及 UTM 界面、Ubuntu 镜像和校验流程的信息于 **2026-07-17** 根据官方文档核对。UTM 的按钮名称、虚拟化后端和 Ubuntu 点版本会变化，实际操作时应重新核对文末官方资料。

## 完成标准

完成本文后，应能够：

- 根据宿主机 CPU 架构选择正确的 Ubuntu Server ISO。
- 解释 UTM 中 **Virtualize** 与 **Emulate** 的差异，并选择合适路线。
- 验证 ISO 文件名、文件类型和 SHA-256 摘要。
- 在 UTM 中创建虚拟机并完成 Ubuntu Server 安装。
- 安装结束后卸载 ISO，确认系统从虚拟磁盘启动。
- 在客户机内验证架构、系统版本、磁盘、网络、DNS 和时间状态。
- 遇到 EFI Shell、重复进入安装器、网络不可用等问题时，知道从哪一层排查。

## 1. 先确定使用 Virtualize 还是 Emulate

UTM 同时支持虚拟化和模拟，但二者解决的问题不同。

| 模式 | 宿主机与客户机架构 | 适用场景 | 代价 |
| --- | --- | --- | --- |
| Virtualize | 架构需要匹配 | 日常 Linux 开发机、构建和容器实验 | 性能接近原生，主线优先 |
| Emulate | 可以不同 | 必须运行另一种 CPU 架构的旧系统或软件 | 需要指令翻译，通常更慢、耗能更多 |

UTM 官方说明：虚拟化要求客户机架构与宿主机匹配。Apple Silicon 宿主机对应 `aarch64` / ARM64，Intel Mac 对应 `x86_64` / AMD64。如果架构不匹配，即使勾选了虚拟化，UTM 也不能按同架构虚拟化执行。

先在宿主机确认架构。

**执行位置：macOS 宿主机（任意目录）**

```bash
uname -m
sysctl -n hw.logicalcpu
sysctl -n hw.memsize
```

常见结果：

- `arm64`：选择 Ubuntu Server ARM64 ISO，并走 **Virtualize → Linux**。
- `x86_64`：选择 Ubuntu Server AMD64 ISO，并走 **Virtualize → Linux**。

只有在明确需要跨架构系统时才选择 **Emulate**。后端开发工具和容器镜像如果支持宿主架构，通常没有必要先引入模拟层。

## 2. 选择 Ubuntu 版本与镜像

Ubuntu 版本不是“越新越好”。应按以下顺序决策：

1. 项目或团队已声明的操作系统约束。
2. 仍在安全维护期内的 Ubuntu Server LTS。
3. 所需 JDK、Go、Docker、数据库和其他原生依赖对该版本的支持情况。
4. 宿主机 CPU 架构。

从 Ubuntu 官方下载目录获取 Server 安装 ISO。文件名通常会包含：

- Ubuntu 版本。
- `live-server`。
- `arm64` 或 `amd64`。
- `.iso`。

不要只根据“64 位”判断架构，因为 ARM64 和 AMD64 都是 64 位指令集。

## 3. 校验 ISO 架构与完整性

下载 ISO 后，同时下载同一发布目录中的 `SHA256SUMS` 和 `SHA256SUMS.gpg`。先用下面脚本确认宿主机架构与 ISO 文件名相符，再计算实际摘要。

**执行位置：macOS 宿主机（任意目录）**

```bash
printf 'Ubuntu Server ISO 绝对路径: '
IFS= read -r ISO_FILE

if [ ! -f "$ISO_FILE" ]; then
  printf '停止：文件不存在：%s\n' "$ISO_FILE" >&2
  exit 1
fi

HOST_ARCH="$(uname -m)"
ISO_NAME="$(basename "$ISO_FILE")"

case "$HOST_ARCH" in
  arm64)
    EXPECTED_ARCH_TOKEN='arm64'
    ;;
  x86_64)
    EXPECTED_ARCH_TOKEN='amd64'
    ;;
  *)
    printf '停止：尚未为宿主机架构 %s 选择默认镜像。\n' "$HOST_ARCH" >&2
    exit 1
    ;;
esac

case "$ISO_NAME" in
  *"$EXPECTED_ARCH_TOKEN"*.iso)
    printf '架构文件名检查通过：host=%s iso=%s\n' "$HOST_ARCH" "$ISO_NAME"
    ;;
  *)
    printf '停止：ISO 文件名与宿主机架构不匹配：host=%s iso=%s\n' \
      "$HOST_ARCH" "$ISO_NAME" >&2
    exit 1
    ;;
esac

file "$ISO_FILE"
shasum -a 256 "$ISO_FILE"
```

预期结果：

- 文件名包含宿主机对应的架构标记。
- `file` 将文件识别为 ISO 9660 或可启动光盘映像。
- `shasum` 输出 64 位十六进制 SHA-256 摘要。

再把实际摘要与官方 `SHA256SUMS` 中同名文件的摘要比较。

**执行位置：macOS 宿主机（ISO 与 `SHA256SUMS` 位于同一目录）**

```bash
printf 'Ubuntu Server ISO 绝对路径: '
IFS= read -r ISO_FILE

ISO_DIR="$(dirname "$ISO_FILE")"
ISO_NAME="$(basename "$ISO_FILE")"
CHECKSUM_FILE="$ISO_DIR/SHA256SUMS"

if [ ! -f "$ISO_FILE" ] || [ ! -f "$CHECKSUM_FILE" ]; then
  printf '%s\n' '停止：ISO 或 SHA256SUMS 不存在。' >&2
  exit 1
fi

EXPECTED_HASH="$(
  awk -v name="$ISO_NAME" \
    '$2 == name || $2 == "*" name { print $1; exit }' \
    "$CHECKSUM_FILE"
)"
ACTUAL_HASH="$(shasum -a 256 "$ISO_FILE" | awk '{ print $1 }')"

printf 'expected=%s\nactual=%s\n' "$EXPECTED_HASH" "$ACTUAL_HASH"

if [ -n "$EXPECTED_HASH" ] && [ "$EXPECTED_HASH" = "$ACTUAL_HASH" ]; then
  printf '%s\n' 'SHA-256 校验通过。'
else
  printf '%s\n' '校验失败：不要启动该 ISO，请重新核对下载来源。' >&2
  exit 1
fi
```

如果 `EXPECTED_HASH` 为空，通常是 ISO 被重命名，或 ISO 与校验文件不属于同一发布目录。不要手工修改校验清单来“让它通过”。

SHA-256 对比能发现文件损坏或清单不一致；若要验证校验清单本身的真实性，还应按 Ubuntu 官方流程使用 `SHA256SUMS.gpg` 和 Ubuntu Image Signing Key 建立签名信任链。

## 4. 创建虚拟机

### 4.1 创建前记录决策

先在 [[UTM 虚拟机网络与资源规划]] 中明确：

- 客户机架构与 UTM 模式。
- vCPU、内存和磁盘上限。
- Shared Network、Bridged Network 或其他网络模式。
- 是否需要共享目录。
- 虚拟机名称和用途。

资源值应根据宿主机容量和工作负载选择，不要把示例规格照抄成唯一答案。为宿主机保留足够 CPU、内存和磁盘空间，并考虑虚拟磁盘增长、Docker 镜像和后续备份占用。

### 4.2 使用创建向导

UTM 不同版本的文字和顺序可能略有变化，核心步骤如下：

1. 打开 UTM，选择新建虚拟机。
2. 选择 **Virtualize**。
3. 选择 **Linux**。
4. 选择已经校验通过的 Ubuntu Server ISO。
5. 按资源规划填写 vCPU 和内存。
6. 设置虚拟磁盘容量上限。
7. 按网络规划选择网络模式。
8. 默认先不配置项目工作区共享目录。
9. 保存虚拟机配置，并在启动前再次核对架构、ISO 和磁盘。

开发仓库长期放在 Linux 本地文件系统更接近服务器真实语义。共享目录适合临时交换安装文件或导出物，不应在未验证权限、文件监听、符号链接和性能行为前，直接作为主要开发目录。

> [!tip] 创建后先不要急着启动
> 重新打开虚拟机设置，确认 ISO 架构、启动介质、CPU、内存、磁盘和网络与规划一致。此时修正配置的成本最低。

## 5. 安装 Ubuntu Server

启动虚拟机后进入 Ubuntu Server 安装器。安装界面会随 Ubuntu 版本变化，但应逐项理解以下选择。

### 5.1 语言、键盘与网络

- 选择能够稳定操作的语言和键盘布局。
- 主线使用 DHCP 自动获取地址。
- 没有明确代理时，不要随意填写来源不明的代理地址。
- 软件源使用 Ubuntu 官方默认配置；网络受限时先诊断 DNS、时间和代理，不要直接替换为不明镜像。

安装器内显示的地址通常由 DHCP 分配，不应写成长期固定资产。后续以客户机中的 `ip address`、DNS 解析和 SSH 实测为准。

### 5.2 存储

对单用途开发虚拟机，可以使用安装器的引导式整盘方案，让 Ubuntu 使用 UTM 提供的虚拟磁盘。这里操作的是虚拟磁盘，不是宿主机真实系统盘。

提交分区方案前检查：

- 安装目标确实是预期虚拟磁盘。
- 容量与 UTM 中设置的磁盘上限相符。
- 没有误选安装 ISO 或额外数据盘。

> [!warning] 分区确认会覆盖目标虚拟磁盘上的原有内容
> 新建空虚拟机通常没有需要保留的数据；重装现有虚拟机前必须先建立 [[UTM 虚拟机快照、备份与恢复]] 中的独立备份。

### 5.3 用户、主机名与 SSH

- 创建普通管理员用户，不要直接以 `root` 作为日常账号。
- 主机名应简短、稳定、只表达机器用途，不包含密码、IP 或人员隐私。
- 使用唯一且足够强的密码完成初次安装；后续再配置 SSH 密钥。
- 如果安装器提供 OpenSSH Server 选项，可以安装服务端；不要在安装界面导入未经核对的第三方公钥。

用户、`sudo`、权限和安全初始化在 [[Ubuntu Server 初始化与基础安全]] 中继续完成；SSH 客户端、服务端、密钥和主机指纹见 [[OpenSSH 连接、密钥与主机指纹]]。

### 5.4 等待安装完成

等待安装器明确显示安装完成后再重启。安装期间不要强制关闭 UTM；强制关机会丢弃未写入数据，甚至破坏虚拟磁盘文件系统。

## 6. 卸载 ISO 并从虚拟磁盘启动

安装结束后的第一次重启可能因为 ISO 仍挂载而再次进入安装器，或停在黑屏。UTM 官方 Ubuntu 指南建议：若安装完成后未自动正常重启，可以关闭虚拟机、卸载安装 ISO，再重新启动。

操作顺序：

1. 确认安装器已经明确完成写入。
2. 关闭虚拟机窗口或让客户机正常停止。
3. 在 UTM 的可移动驱动器或 CD/DVD 菜单中卸载 ISO。
4. 检查启动设备仍包含系统虚拟磁盘。
5. 再次启动虚拟机。

判断是否成功：

- 启动过程不再进入 Ubuntu 安装器。
- 出现已安装系统的登录提示。
- 使用安装时创建的普通用户可以登录。

如果仍进入安装器，先检查 ISO 是否确实卸载，再检查启动顺序。不要重复安装覆盖已经写入的系统。

## 7. 首次启动验证

登录客户机控制台后，先做只读验证，不要立即安装大量工具。

**执行位置：Ubuntu 虚拟机（UTM 控制台，任意目录）**

```bash
printf '%s\n' 'Architecture:'
uname -m
dpkg --print-architecture

printf '\n%s\n' 'Operating system:'
cat /etc/os-release

printf '\n%s\n' 'Root filesystem and block devices:'
findmnt /
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
df -h /

printf '\n%s\n' 'Network and route:'
ip -brief address
ip route

printf '\n%s\n' 'DNS and time:'
getent hosts ubuntu.com
timedatectl

printf '\n%s\n' 'Failed systemd units:'
systemctl --failed
```

预期结果：

- `uname -m` 与 `dpkg --print-architecture` 对应所选 ISO。
- `/etc/os-release` 显示计划安装的 Ubuntu 系列。
- 根文件系统挂载在虚拟磁盘上，容量符合规划。
- 网卡有地址，并存在默认路由。
- `getent hosts ubuntu.com` 能解析域名。
- 时间同步状态合理。
- `systemctl --failed` 没有未解释的失败单元。

若 IP 地址、DNS 或时间异常，不要先假设“Ubuntu 安装坏了”。按 [[UTM 虚拟机网络与资源规划]] 从 UTM 虚拟网卡、DHCP、客户机接口、默认路由、DNS 和时间逐层检查。

## 8. 常见故障与恢复

| 现象 | 优先检查 | 恢复方式 |
| --- | --- | --- |
| 启动后进入 UEFI Interactive Shell | ISO 是否挂载、是否为正确架构、是否存在可移动驱动器 | 重新挂载已校验 ISO；按 UTM 官方指南检查 EFI 启动项 |
| 提示 `Exec format error` 或无法启动内核 | 宿主机、ISO 和 VM 架构是否一致 | 改用匹配架构的 ISO；跨架构需求单独使用 Emulate |
| SHA-256 校验失败 | ISO 与 `SHA256SUMS` 是否来自同一目录 | 删除损坏下载并从官方来源重新获取 |
| 安装完成后又进入安装器 | ISO 未卸载或启动顺序错误 | 关机、卸载 ISO、让虚拟磁盘优先启动 |
| 安装完成后重启停在黑屏 | ISO、显示设备或启动状态 | 按 UTM Ubuntu 指南关闭 VM、卸载 ISO后重新启动 |
| 客户机没有地址 | UTM 网卡、网络模式、客户机接口 | `ip link`、`ip route`；确认 UTM 虚拟网卡启用 |
| 能访问 IP，不能解析域名 | DNS 或时间 | `resolvectl status`、`getent hosts`、`timedatectl` |
| 宿主机明显卡顿 | vCPU、内存或真实磁盘空间分配过激 | 完全关机后降低资源，检查宿主与客户机空间 |
| 虚拟磁盘容量不够 | 初始规划不足、缓存或镜像增长 | 先备份，再按 UTM 官方磁盘扩容流程扩大并扩展客户机文件系统 |

### 安装失败时如何重新开始

如果安装从未完成，且虚拟磁盘中没有需要保留的数据，可以：

1. 关闭虚拟机。
2. 修正 ISO、架构、磁盘或网络配置。
3. 重新挂载已校验 ISO。
4. 重新运行安装器。

> [!danger] 从 UTM 列表移除虚拟机可能同时删除其数据
> UTM 官方 Actions 文档说明：位于默认路径的 VM 被移除时，数据也可能被删除。只有确认虚拟磁盘中没有需要保留的系统、源码或数据，并且独立备份可用时，才能删除 VM。不要把“移出列表”和“仅取消显示”视为同一操作。

如果系统已投入使用，应先按 [[UTM 虚拟机快照、备份与恢复]] 创建独立副本，再决定修复、重装或回退。

## 9. 安装完成后的下一步

虚拟机能够启动只代表操作系统安装完成，还不是可用的开发环境。下一步应按顺序完成：

1. [[Ubuntu Server 初始化与基础安全]]
2. [[OpenSSH 连接、密钥与主机指纹]]
3. [[Linux 开发工作区与本地文件系统规划]]
4. 项目所需的 Git、语言工具链与容器工具
5. [[UTM 虚拟机快照、备份与恢复]]

## 官方资料

- [UTM Documentation：Ubuntu 指南](https://docs.getutm.app/guides/ubuntu/)
- [UTM Documentation：QEMU System 与架构](https://docs.getutm.app/settings-qemu/system/)
- [UTM Documentation：Actions](https://docs.getutm.app/basics/actions/)
- [UTM Documentation：Controls](https://docs.getutm.app/basics/controls/)
- [UTM Documentation：Resize and Compress](https://docs.getutm.app/settings-qemu/drive/resize-and-compress/)
- [Ubuntu 官方镜像完整性验证](https://documentation.ubuntu.com/security/software-integrity/image-verification/)
- [Ubuntu Server 下载](https://ubuntu.com/download/server)
