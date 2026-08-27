---
title: Git 凭据、SSH 与常见问题排查
aliases:
  - Git SSH 配置
  - Git HTTPS 凭据管理
  - Git 远程认证排障
tags:
  - Git
  - Git/认证
  - Git/SSH
  - Git/HTTPS
  - Git/排障
created: 2026-07-14T22:52:26
updated: 2026-08-26T17:11:22
---

本文说明 Git 已经安装并完成第一次本地提交后，如何通过 HTTPS 或 SSH 安全访问远程仓库，以及如何按连接阶段定位凭据、主机身份、网络和仓库权限问题。本文以 GitHub 为主要示例，但“远程 URL 选择协议、客户端保护秘密、平台验证账号、仓库权限决定读写”的边界同样适用于 GitLab、Gitea 和公司自建平台。

先完成 [[Git 常用配置与本地验证#5. 用独立练习仓库完成第一次本地提交|第一次本地提交验证]]。这里不要求提前完成该笔记末尾的远程协作配置；本篇会先建立远程访问模型。若 Git 命令不存在或版本不符合要求，请先返回 [[Ubuntu 从零安装 Git]] 或 [[macOS 从零安装 Git]]。

> [!important] 提交身份、远程认证与仓库授权是三件事
> `user.name` 和 `user.email` 决定提交历史显示的作者信息；HTTPS 令牌或 SSH 密钥用于向代码平台证明账号身份；平台再根据账号和仓库规则决定能否读取或推送。前一层成功，不能自动证明后一层也成功。

> [!info] 核对日期与适用范围
> 本文于 **2026-08-26** 核对 Git 官方 `gitcredentials`、`git-remote`、`git-ls-remote` 手册，OpenSSH 的 `ssh-agent`、`ssh_config` 手册，以及 GitHub 的 HTTPS、SSH、macOS Keychain 与主机指纹文档。凭据帮助程序、平台登录流程、系统会话提供的 agent 和支持的密钥类型可能变化，应结合本机 `git --version`、`git help`、OpenSSH 手册和目标平台当前文档重新确认。

## 完成标准

完成本篇后，应能区分并验证下列状态。

| 类别 | 应达到的状态 | 不能据此声称什么 |
| --- | --- | --- |
| 远程模型 | 能解释本地仓库、远程 URL、`origin`、网络、凭据和平台权限的关系 | 不代表已经配置任何账号 |
| 协议选择 | 明确当前仓库使用 HTTPS 还是 SSH，并只配置所选路径 | 不代表另一条协议也可用 |
| 账号认证 | HTTPS 登录完成，或 SSH 测试显示预期平台账号 | 不代表对任意仓库都有权限 |
| SSH 日常使用 | 能区分私钥文件、客户端配置、agent 进程和口令存储，并选择当前系统适用的持久方式 | 不代表解密后的密钥应永久驻留内存 |
| 仓库读取 | `git ls-remote` 能与目标仓库通信并读取远程引用 | 不代表拥有推送权限 |
| 安全边界 | 令牌和私钥未进入远程 URL、仓库、笔记、截图或日志 | 不代表泄露后的凭据已经撤销 |

## 1. 先建立远程访问模型

### 1.1 一次远程访问要依次经过哪些层

Git 的远程访问可以先简化为：

```text
本地 Git 仓库或准备 clone 的目录
  → 远程 URL 决定使用 HTTPS 还是 SSH
  → DNS、路由、代理或防火墙决定网络路径是否可达
  → HTTPS 凭据或 SSH 密钥证明平台账号身份
  → 平台根据仓库权限决定能否读取或推送
```

这五层应分别判断：

- **远程仓库**是代码平台或另一台服务器上的独立 Git 仓库。
- **远程名称**是本地为某个远程仓库起的名字；`origin` 只是 `git clone` 默认使用的常见名称，不是 GitHub 的同义词。
- **远程 URL**同时描述目标仓库和访问协议，例如 `https://github.com/ACCOUNT/REPOSITORY.git` 或 `git@github.com:ACCOUNT/REPOSITORY.git`。
- **认证（authentication）**回答“平台把当前凭据识别成哪个账号”。
- **授权（authorization）**回答“这个账号对目标仓库能做什么”。公开仓库可能无需登录即可读取，但推送仍需要写权限。

网络可达不等于认证成功，认证成功也不等于有仓库写权限。排障时应找到第一个失败层，而不是反复更换密钥或重装 Git。

### 1.2 首次 clone 与已有本地仓库是两个起点

| 起点 | 含义 | 后续动作 |
| --- | --- | --- |
| 尚无本地仓库 | 准备从平台取得一个已有仓库 | 配置所选协议后执行 `git clone`；Git 默认创建本地目录并添加 `origin` |
| 已有本地仓库，没有 `origin` | 例如通过 `git init` 创建并完成了本地提交 | 先验证远程 URL，再用 `git remote add origin` 关联 |
| 已有 `origin` | 当前仓库已经关联远程，可能只需认证或切换协议 | 先读取现有 fetch/push URL；验证新 URL 后再用 `git remote set-url` 修改 |

本文只为后续认证建立最小命令语义：

| 命令 | 最小含义 |
| --- | --- |
| `git clone` | 从远程创建一个新的本地仓库和工作区 |
| `git fetch` | 获取远程提交和分支状态，不直接改写当前工作区 |
| `git pull` | 先 fetch，再把上游变化整合到当前分支 |
| `git push` | 把本地引用和对象发送到远程，通常需要仓库写权限 |

分支、上游、pull 与 push 的日常使用不在本篇展开，认证完成后再进入 [[Git 分支与 PR 工作流]]。

## 2. 选择 HTTPS 或 SSH

HTTPS 与 SSH 都可以访问同一个仓库，也不会改变提交内容。两者主要区别在于网络协议和客户端怎样向平台证明身份。

- **HTTPS** 使用 HTTP over TLS 访问平台。平台可能让用户在浏览器完成 OAuth（授权某个客户端代表账号访问），也可能把个人访问令牌（Personal Access Token，PAT）作为秘密凭据。Git 通过 **credential helper（凭据帮助程序）** 从系统钥匙串、密钥服务或内存缓存取得和保存凭据。
- **SSH** 使用本机私钥完成签名，平台根据账号中登记的公钥识别用户。**SSH Agent** 是客户端本地进程，可保存已解锁的密钥并代为签名；它不是代码平台，也不会把私钥上传给平台。

| 协议 | 远程 URL 示例 | 认证材料 | 更适合什么情况 |
| --- | --- | --- | --- |
| HTTPS | `https://github.com/ACCOUNT/REPOSITORY.git` | 浏览器 OAuth、个人访问令牌与受认可的 credential helper | 网络限制 SSH、首次接触平台、组织统一使用 HTTPS |
| SSH | `git@github.com:ACCOUNT/REPOSITORY.git` | 本机私钥、平台登记的公钥，可选 SSH Agent | 经常操作远程仓库、已能管理密钥、组织统一使用 SSH |

选择应服从团队规范、组织安全要求和实际网络环境。一次配置先选择一条路径；不要为了“看起来高级”同时改动 HTTPS helper、SSH 密钥和远程 URL。无论选择哪一种，都不要把密码、令牌或私钥写入远程 URL、Shell 配置、Git 仓库、提交消息或 Markdown。

## 3. HTTPS：使用平台认证与安全凭据存储

### 3.1 先理解并检查 credential helper

credential helper 是 Git 为 HTTPS 凭据调用的外部程序。它可能把凭据暂存在进程内存中、保存在操作系统安全存储中，或通过浏览器 OAuth 生成短期凭据。它不参与 SSH 私钥认证。

先查看 Git 当前会按哪些来源调用 helper，以及本机安装了哪些候选程序：

```bash
git config --show-origin --get-all credential.helper
git help -a | grep credential-
```

第一条没有输出，通常表示当前可见配置没有设置 helper；出现多行时，Git 可能依次询问多个 helper，必须结合 `--show-origin` 判断它们来自 system、global 还是当前仓库的 local 配置。第二条没有匹配，只能说明当前 Git 没有发现相应外部命令，不能靠手写一个 helper 名称让它自动安装。

不要覆盖组织管理的 system 或 local 配置。只有确认当前没有可用的受认可 helper，才在下面选择一个与系统匹配的 global 配置；global 会影响当前用户的其他仓库。

### 3.2 macOS：优先使用 Keychain

若 `git help -a` 确实列出 `credential-osxkeychain`，且现有配置没有组织指定的其他 helper，可以添加 macOS Keychain：

```bash
git config --global --add credential.helper osxkeychain
git config --show-origin --get-all credential.helper
```

验证输出应出现来自 global 配置的 `osxkeychain`。随后第一次访问需要认证的 HTTPS 远程时，按平台要求输入账号名和令牌；`osxkeychain` 负责从 Keychain 读取和保存凭据，本身不负责创建令牌或发起浏览器 OAuth。若组织使用专门的 OAuth helper，则按该 helper 的浏览器流程完成登录。

如果这一值确实是刚按本节新增、且需要撤销，只移除这个精确 global 值，再重新检查全部来源：

```bash
git config --global --unset-all credential.helper '^osxkeychain$'
git config --show-origin --get-all credential.helper
```

### 3.3 Ubuntu：使用可用的安全存储或组织方案

Linux 上常见的安全持久化 helper 是 `libsecret`，它可与 GNOME Keyring、KDE Wallet 等 Secret Service 实现集成。只有 `git help -a` 确实列出 `credential-libsecret`，且桌面密钥服务与组织规则允许时，才添加：

```bash
git config --global --add credential.helper libsecret
git config --show-origin --get-all credential.helper
```

若它不存在，应使用组织批准的 Git Credential Manager、OAuth helper，或改走 SSH；不要因为图省事改用明文存储。需要撤销本节刚添加的值时：

```bash
git config --global --unset-all credential.helper '^libsecret$'
git config --show-origin --get-all credential.helper
```

### 3.4 仅在明确需要时使用临时内存缓存

在安全持久化存储不可用、且确实只需要短时认证时，可把凭据缓存约一小时：

```bash
git config --global --add credential.helper 'cache --timeout=3600'
git config --show-origin --get-all credential.helper
```

缓存到期、helper 进程退出或系统重启后需要重新认证。它不应替代组织要求的凭据方案。若该值是本节刚添加的，可精确撤销：

```bash
git config --global --unset-all credential.helper '^cache --timeout=3600$'
git config --show-origin --get-all credential.helper
```

> [!danger] 不要使用明文 `store` 保存长期令牌
> `git config --global credential.helper store` 会把凭据以未加密形式写入磁盘，只依靠文件权限保护。优先使用系统安全存储、受认可的 OAuth/helper 方案或 SSH 密钥。

HTTPS 的实际登录会在第 5 节访问目标 URL 时触发。令牌应只输入平台或受信任 helper 提供的认证界面，不得把用户名和令牌直接拼进远程 URL。

## 4. SSH：使用专用密钥访问代码平台

### 4.1 先分清主机身份、账号身份和仓库权限

SSH 访问托管代码平台时存在三次不同判断：

1. SSH 客户端根据主机公钥和 `known_hosts` 判断连接的是否是预期代码平台。
2. 平台让客户端用用户私钥签名，并根据账号中登记的公钥识别平台账号。
3. 平台根据该账号和仓库策略决定读取或推送权限。

用户私钥始终留在客户端；平台只登记对应公钥。托管平台的“账号 SSH key”不是让用户登录平台服务器的普通 Linux `authorized_keys` 操作，不要照搬自建服务器配置。

主机身份、用户身份、私钥与公钥的通用模型见 [[OpenSSH 密钥登录、服务端配置与排查#1. 先建立够用的 SSH 连接模型]]；`ssh`、`ssh-keygen`、`ssh -G`、`-i` 与 `-vvv` 的命令边界见 [[OpenSSH 常用命令基础]]。本篇只保留 Git 平台需要的操作顺序和验证语义。

### 4.2 先检查已有密钥

`test -d` 只检查目录是否存在，下面的 `if` 根据检查结果选择输出，不会创建或修改文件：

```bash
if test -d "$HOME/.ssh"; then
  ls -ld "$HOME/.ssh"
  ls -l "$HOME/.ssh"
else
  printf '%s\n' '当前用户尚无 .ssh 目录。'
fi
```

常见公钥文件以 `.pub` 结尾，例如 `id_ed25519.pub`。同名但没有 `.pub` 的文件通常是私钥，绝不能上传、发送、粘贴或提交。有现成密钥不代表它已登记到当前平台，也不代表它仍符合用途隔离和组织策略；先确认用途，再决定复用或创建专用密钥。

### 4.3 使用统一路径安全创建专用密钥

下面以 `$HOME/.ssh/id_ed25519_github` 为 GitHub 专用私钥路径。先执行只读检查：`test -L` 也会识别失效的符号链接，避免把已有链接当作空闲路径。

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

if test -L "$HOME/.ssh"; then
  printf '%s\n' '停止：.ssh 是符号链接，请先确认其真实目标和安全边界。'
elif test -e "$HOME/.ssh" && ! test -d "$HOME/.ssh"; then
  printf '%s\n' '停止：.ssh 已存在但不是目录。'
elif test -e "$KEY_PATH" || test -L "$KEY_PATH" ||
     test -e "$KEY_PATH.pub" || test -L "$KEY_PATH.pub"; then
  printf '停止：密钥路径已占用，请确认用途并选择新名称：%s\n' "$KEY_PATH"
else
  printf '路径可用，可以创建专用密钥：%s\n' "$KEY_PATH"
fi
```

只有最后看到“路径可用”时才继续。以下命令会创建 `.ssh` 目录、私钥和同名公钥，并修改这些新对象的权限：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

mkdir -p "$HOME/.ssh" &&
  chmod 700 "$HOME/.ssh" &&
  ssh-keygen -t ed25519 -a 64 -f "$KEY_PATH" -C 'github-access' &&
  chmod 600 "$KEY_PATH" &&
  chmod 644 "$KEY_PATH.pub" &&
  ssh-keygen -lf "$KEY_PATH.pub"
```

`&&` 表示上一条命令成功后才继续；任一步失败都应停止并检查已经创建的对象，不要跳过失败直接执行后续权限命令。`-C` 只写入帮助识别用途的注释，不是提交身份或平台密码。为私钥设置强口令；不要用 `sudo` 创建当前用户的密钥，也不要递归修改整个 HOME。`ssh-keygen` 参数与防覆盖边界的通用解释见 [[OpenSSH 常用命令基础#3.1 创建用户密钥]]。

创建后可以只读复核：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

ls -ld "$HOME/.ssh"
ls -l "$KEY_PATH" "$KEY_PATH.pub"
ssh-keygen -lf "$KEY_PATH.pub"
```

### 4.4 检查 SSH Agent，再选择临时恢复或日常配置

SSH Agent 是客户端本地进程，用于在一段会话生命周期内缓存可用身份，避免每次使用有口令的私钥都重新解锁。它不会替代磁盘上的私钥文件，也不会替代 `~/.ssh/config` 中的密钥选择规则。

需要先区分三种容易混称为“持久化”的状态：

| 状态 | 负责对象 | 跨新终端或重新登录后的边界 |
| --- | --- | --- |
| 私钥文件持久存在 | `$HOME/.ssh/id_ed25519_github` | 文件仍在，但 SSH 不一定会自动选择自定义路径 |
| 客户端持续选择该文件 | `IdentityFile`、`IdentitiesOnly` | `~/.ssh/config` 仍在时继续生效，不等于 agent 已运行或口令已缓存 |
| 解锁后的身份暂存于 agent | agent 进程、通信 socket 和已加载身份 | 只在相应 agent 生命周期内可用；是否跨终端或登录取决于系统会话怎样提供 agent |

`SSH_AUTH_SOCK` 指向当前 Shell 怎样联系 agent，`ssh-add -l` 列出 agent 已加载密钥的指纹，不会显示私钥内容。

#### 4.4.1 先检查当前登录会话

```bash
printf 'SSH_AUTH_SOCK=%s\n' "${SSH_AUTH_SOCK:-未设置}"
ssh-add -l -E sha256
```

结果应分别解释：

- 能列出指纹：agent 可用，并已加载至少一把密钥。
- 显示没有身份（no identities）：agent 可用，但尚未加载密钥。
- 显示无法连接 agent：当前 Shell 没有可用 agent；不要用 `2>/dev/null` 隐藏这个区别。

Ubuntu 桌面、终端管理器、远程登录会话或 macOS 登录会话可能已经提供 agent。只要当前 agent 可用，就复用它，不要再启动第二个进程。

#### 4.4.2 当前没有 agent：只做临时恢复

只有确认当前 Shell 无法连接 agent 时才进入本节。先确认当前 Shell 实际解析到哪个 `ssh-agent`：

```bash
type -a ssh-agent
```

输出可能因系统与安装方式不同。若名称被别名、函数或用途不明的程序遮蔽，先确认来源，不要直接执行其生成的代码。确认使用可信 OpenSSH `ssh-agent` 后，才临时启动：

```bash
eval "$(ssh-agent -s)"
```

这行命令包含三个步骤：`ssh-agent -s` 启动后台 agent 并输出设置通信变量的 Shell 语句；`$()` 捕获这些输出；`eval` 再让**当前 Shell**解析并执行这些语句。命令替换与 `eval` 的区别、安全边界见 [[Shell 路径、变量、引用与展开#6.1 eval 会把文本交给当前 Shell 再解析一次|eval 的二次解析边界]]。这里只能执行已经确认来源的 `ssh-agent` 输出，不得把用户输入、网络响应或任意变量替换到 `eval` 中。

Ubuntu 中可以随后加载本篇使用的同一路径并复核指纹：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

ssh-add "$KEY_PATH" &&
  ssh-add -l -E sha256
```

`eval` 设置的 `SSH_AUTH_SOCK` 和 `SSH_AGENT_PID` 只会由当前 Shell 及其后续子进程继承；无关的新 Shell 不会自动取得这组变量。后台 agent 也不保证在当前终端关闭时立即退出，所以这只是临时恢复，不是日常持久方案。不得把这条启动命令无条件写入 `.bashrc` 或 `.zshrc`，否则每个新 Shell 都可能启动新的 agent。

如果这个 agent 确实由本节临时启动，且已经不再使用，可以停止它并清理当前 Shell 的通信变量：

```bash
ssh-agent -k &&
  unset SSH_AUTH_SOCK SSH_AGENT_PID
```

不要用这条命令停止桌面环境、登录会话、IDE 或其他工具统一管理的 agent。

#### 4.4.3 日常使用：持久选择密钥并复用会话 agent

本篇使用的是自定义文件名 `id_ed25519_github`，不应依赖它碰巧仍在某个 agent 中。先只读检查已有用户级 SSH 配置：

```bash
SSH_CONFIG="$HOME/.ssh/config"

if test -e "$SSH_CONFIG" || test -L "$SSH_CONFIG"; then
  ls -l "$SSH_CONFIG"
  sed -n '1,240p' "$SSH_CONFIG"
else
  printf '%s\n' '当前用户尚无 SSH 客户端配置。'
fi
```

若文件已经存在，应先备份或记录原内容，再把对应配置合并到正确的 `Host` 块；不得覆盖用途未知的配置，也不要重复追加同名块。具体读取顺序、字段含义和“具体规则在前、通用规则在后”的边界见 [[OpenSSH 常用命令基础#2.6 使用 ~/.ssh/config 别名]]。

Ubuntu 或其他不使用 macOS Keychain 的客户端，可以使用以下 GitHub 配置；其他平台应把 `Host` 改为实际 SSH 主机或已经规划好的本地别名：

```sshconfig
Host github.com
    AddKeysToAgent yes
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
```

- `IdentityFile` 让后续新终端和新登录仍能找到自定义路径中的私钥。
- `IdentitiesOnly yes` 限制 SSH 使用明确配置的候选身份，避免 agent 中其他密钥干扰账号判断。
- `AddKeysToAgent yes` 只会在 SSH 从文件成功读取密钥时，把它加入**已经运行的 agent**；它不会启动 agent，也不会把解锁后的密钥永久写入配置。

保存后先解析配置，不建立网络连接：

```bash
ssh -G git@github.com |
  grep -E '^(hostname|user|identityfile|identitiesonly|addkeystoagent) '
```

输出中应包含 `github.com`、`git`、本篇自定义私钥路径和 `identitiesonly yes`，并显示 `addkeystoagent` 已启用；不同 OpenSSH 版本可能把启用值规范化输出为 `yes` 或 `true`。若结果与预期不同，先恢复或修正刚才合并的配置，不要继续真实连接。`ssh -G` 仍会读取 `Include`，可信配置中的 `Match exec` 也可能执行本地命令。

#### 4.4.4 macOS：同时使用系统 agent 与 Keychain

macOS 若要在新登录后继续自动选择私钥，并由系统 Keychain 保存私钥口令，应在同一个 `Host` 块使用下面这一份配置，不要再额外保留上一节的重复块：

```sshconfig
Host github.com
    IgnoreUnknown UseKeychain
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
```

`IgnoreUnknown UseKeychain` 必须出现在 `UseKeychain` 之前，便于同一份配置在不认识苹果扩展的 OpenSSH 上跳过该字段。`UseKeychain yes` 保存和读取的是私钥口令，私钥文件仍留在 `$HOME/.ssh`。确认当前 agent 可用后，使用苹果系统自带的 `ssh-add` 完成一次加载和 Keychain 保存：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

/usr/bin/ssh-add --apple-use-keychain "$KEY_PATH" &&
  /usr/bin/ssh-add -l -E sha256
```

`--apple-use-keychain` 只适用于苹果系统自带的 `ssh-add`，不得复制到 Ubuntu。若私钥没有口令，应省略 `UseKeychain` 和 `--apple-use-keychain`；本篇仍建议为普通磁盘私钥设置强口令。

#### 4.4.5 Ubuntu：持久程度取决于登录会话

Ubuntu 登录会话若已经提供 agent，上一节的 `IdentityFile` 与 `AddKeysToAgent yes` 可以避免每个终端都手工执行 `ssh-add`：第一次实际使用私钥时可能需要输入一次口令，随后同一 agent 生命周期内可以复用。

重新登录或重启后是否仍有 agent、是否自动解锁密钥，取决于桌面环境、终端管理器、密钥环或用户会话服务。若新登录后第 4.4.1 节仍显示无法连接 agent，先使用第 4.4.2 节临时恢复；需要跨登录管理 agent 时，应针对实际 Ubuntu 版本和会话管理器设计用户级服务，不能把某台机器的 socket 路径或启动脚本当成通用答案。本篇不展开 systemd 用户服务，也不建议通过移除私钥口令来换取便利。

### 4.5 将公钥登记到平台

先核对指纹，再显示并复制**公钥**的单行内容：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

ssh-keygen -lf "$KEY_PATH.pub" &&
  cat "$KEY_PATH.pub"
```

然后在代码平台的账号安全设置中新增 SSH key。GitHub 当前入口和步骤见 [GitHub：添加新的 SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)。GitLab、Gitea 或公司平台应使用自己的当前文档。

不得运行 `cat "$KEY_PATH"`，也不得把私钥上传到平台。登记公钥只建立“密钥可能映射到账号”的条件，实际认证仍须下一节的新连接验证。

### 4.6 核对主机身份并验证预期账号

先只读检查本次连接会采用的目标、用户和身份文件：

```bash
ssh -G git@github.com |
  grep -E '^(hostname|user|port|identityfile|identitiesonly) '
```

`ssh -G` 不建立 SSH 网络连接，但可信配置中的 `Match exec` 可能执行本地命令；完整边界见 [[OpenSSH 常用命令基础#2.6 使用 ~/.ssh/config 别名]]。

以 GitHub 和本篇专用密钥为例，显式限定身份后发起测试：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

ssh -T -o IdentitiesOnly=yes -i "$KEY_PATH" git@github.com
```

首次连接时，先核验终端显示的密钥类型和主机指纹与 [GitHub 官方 SSH 指纹](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints) 一致，再接受。主机指纹验证的是“连接到谁”，不是用户公钥是否登记成功；通用原理和主机身份变化恢复见 [[OpenSSH 密钥登录、服务端配置与排查#10. 安全处理主机身份变化警告]]。

GitHub 成功时会显示账号名以及“不提供 shell access”的消息，远程测试仍预期返回状态码 1；应按平台官方消息判断，而不是只把非零状态当作失败。确认消息中的账号正是预期账号。其他平台的 SSH 用户名、测试命令和成功语义可能不同，必须使用对应平台文档。

## 5. 使用远程 URL 接入并验证仓库读取

从代码平台复制目标仓库的 HTTPS 或 SSH URL，不要手工猜测账号名、仓库名或企业域名。正常 URL 不应包含密码、令牌或私钥内容。

### 5.1 改变本地配置前，先验证候选 URL

`git ls-remote` 可以在任意目录直接读取候选 URL 的远程引用，不会创建本地仓库，也不会修改现有工作区：

```bash
# 二选一：保留与第 2 节所选协议一致的赋值。
REMOTE_URL='https://github.com/ACCOUNT/REPOSITORY.git'
# REMOTE_URL='git@github.com:ACCOUNT/REPOSITORY.git'

git ls-remote "$REMOTE_URL"
```

把示例替换为平台实际提供的 URL。HTTPS 私有仓库可能在这里触发 OAuth 或令牌输入；SSH 路径会使用前面配置的密钥。命令成功但没有输出，可能只是远程仓库尚无引用；应结合退出状态和平台仓库状态判断。

成功只证明当前 URL 可以读取远程引用。对于公开 HTTPS 仓库，这一步可能根本没有使用个人凭据；无论哪种协议，它都不能证明拥有推送权限。

### 5.2 尚无本地仓库：安全完成首次 clone

`git clone` 会创建目标目录并写入文件。先选择一个真实存在的父目录，并确认目标路径尚未被文件、目录或符号链接占用。下面的 `test -e`/`test -L` 只读取路径状态，`if` 根据结果选择提示，不会创建或删除对象：

```bash
CLONE_DIR="$HOME/Projects/repository"

if test -e "$CLONE_DIR" || test -L "$CLONE_DIR"; then
  printf '停止：clone 目标已存在，请先检查并选择其他目录：%s\n' "$CLONE_DIR"
else
  printf '目标尚不存在，可以用于 clone：%s\n' "$CLONE_DIR"
fi
```

只有看到“目标尚不存在”，并确认其父目录正确时才继续：

```bash
# 二选一：保留与第 2 节所选协议一致的赋值。
REMOTE_URL='https://github.com/ACCOUNT/REPOSITORY.git'
# REMOTE_URL='git@github.com:ACCOUNT/REPOSITORY.git'
CLONE_DIR="$HOME/Projects/repository"

git clone -- "$REMOTE_URL" "$CLONE_DIR" &&
  cd "$CLONE_DIR" &&
  git remote -v &&
  git status --short --branch
```

成功后通常会出现名为 `origin` 的远程。clone 会创建文件和远程跟踪引用，但不会证明当前账号可以推送。

### 5.3 已有本地仓库：检查、添加或切换 origin

先在目标仓库中只读确认仓库根目录和全部远程：

```bash
git rev-parse --show-toplevel
git remote
git remote -v
```

#### 没有 origin：添加远程

先按第 5.1 节验证候选 URL。成功后再添加：

```bash
# 二选一：保留与第 2 节所选协议一致的赋值。
REMOTE_URL='https://github.com/ACCOUNT/REPOSITORY.git'
# REMOTE_URL='git@github.com:ACCOUNT/REPOSITORY.git'

git ls-remote "$REMOTE_URL" &&
  git remote add origin "$REMOTE_URL" &&
  git remote -v &&
  git ls-remote origin
```

`git remote add` 只修改当前仓库的 Git 配置。若 URL 填错，而且 `origin` 确实是刚由本节新增、尚未执行 fetch 或其他远程操作，可以恢复：

```bash
git remote remove origin
git remote -v
```

不要用 `remove` 处理原本就存在、用途未知的远程。

#### 已有 origin：验证后切换 URL

先读取所有 fetch 与 push URL：

```bash
git remote get-url --all origin
git remote get-url --push --all origin
```

如果存在多个 URL，或 fetch 与 push URL 不同，先停止并确认仓库设计，不要套用下面的单 URL 示例。简单场景先保存旧值、验证新值，再修改：

```bash
# 二选一：保留与第 2 节所选协议一致的赋值。
NEW_REMOTE_URL='https://github.com/ACCOUNT/REPOSITORY.git'
# NEW_REMOTE_URL='git@github.com:ACCOUNT/REPOSITORY.git'
OLD_REMOTE_URL="$(git remote get-url origin)"

printf 'old=%s\nnew=%s\n' "$OLD_REMOTE_URL" "$NEW_REMOTE_URL"
git ls-remote "$NEW_REMOTE_URL" &&
  git remote set-url origin "$NEW_REMOTE_URL" &&
  git remote -v &&
  git ls-remote origin
```

若修改后发现目标错误，且仍在保存 `OLD_REMOTE_URL` 的同一 Shell 中，恢复并复核：

```bash
git remote set-url origin "$OLD_REMOTE_URL" &&
  git remote -v &&
  git ls-remote origin
```

如果原 Shell 已关闭，应使用修改前人工记录并核对过的旧 URL，不要凭记忆猜测。

### 5.4 正确理解验证证据

| 验证 | 能证明什么 | 不能证明什么 |
| --- | --- | --- |
| `ssh -T git@github.com` | GitHub 接受某个 SSH 身份，并显示映射账号 | 该账号对特定仓库有读写权限 |
| `git ls-remote REMOTE_URL` | 当前 URL 能与目标仓库通信并读取远程引用 | 推送权限；公开仓库也可能不需要认证 |
| `git remote -v` | 当前本地保存了哪些 fetch/push URL | 网络、认证或仓库权限是否正常 |
| 后续受控 `git push` | 当前账号对具体引用拥有写入权限 | 提交内容、CI 或评审已经通过 |

本篇以不改动远程仓库的读取验证作为停止点。第一次受控推送放到第 8 节链接的分支工作流中完成。

## 6. 更换、撤销或清理凭据

凭据生命周期涉及三个不同位置，不能只靠一条 `git config --unset` 完成：

1. **代码平台**：撤销泄露、过期或不再需要的令牌，或删除不再使用的账号 SSH 公钥。
2. **本地安全存储或 agent**：从 Keychain、Secret Service、组织凭据管理器或当前 SSH Agent 移除旧凭据。
3. **Git 或 SSH 配置**：检查并移除已经确认不再需要的 helper、`IdentityFile` 或远程 URL 配置。

处理 HTTPS 时，先查看 helper 的全部值和来源，再通过对应系统存储或 helper 的受支持方式删除目标条目：

```bash
git config --show-origin --get-all credential.helper
```

删除一个 helper 配置不会自动删除 Keychain、Secret Service 或平台中的令牌；反过来，撤销平台令牌也不会自动清理本地旧条目。不要使用无目标的 `--unset-all` 清除用途未知的多套 helper。

处理 SSH 时，可以先读取 agent 中的指纹，再只卸载已经确认的私钥：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

ssh-add -l -E sha256
ssh-add -d "$KEY_PATH"
ssh-add -l -E sha256
```

`ssh-add -d` 只从当前 agent 卸载身份，不删除磁盘文件，也不删除平台登记。删除本地私钥前，应确认没有仓库、自动化、备份或恢复流程仍依赖它，并先完成新凭据验证。

若怀疑令牌或私钥已经泄露，先在平台撤销对应凭据，再排查远程 URL、系统凭据存储、Shell 历史、日志、截图和仓库文件。不要把“新建了一个凭据”当作旧凭据已经失效。

## 7. 按第一个失败阶段排查

排障顺序应与访问顺序一致：

```text
远程 URL 与协议
  → 网络、DNS 与代理
  → SSH 主机身份或 HTTPS TLS
  → 客户端提供的密钥、令牌或 helper
  → 平台账号认证
  → 目标仓库授权
```

### 7.1 先确认 URL 和协议

在已有仓库根目录执行：

```bash
git remote -v
git remote get-url --all origin
git remote get-url --push --all origin
```

- `https://...` 进入 HTTPS 路径。
- `git@...` 或 `ssh://...` 进入 SSH 路径。
- 没有 `origin` 时不是认证错误，应回到第 5.3 节确认是否需要添加远程。

不要把包含令牌的 URL、公司内部主机名或代理地址直接复制到公开 Issue、聊天或截图。

### 7.2 SSH：先主机身份，再用户公钥认证

首次连接提示主机真实性无法建立时，先与平台通过独立可信渠道公布的主机指纹核对。若出现 `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`，先停止连接并按 [[OpenSSH 密钥登录、服务端配置与排查#10. 安全处理主机身份变化警告]] 确认变化原因、备份和精确处理旧记录；不得删除整个 `known_hosts` 或设置 `StrictHostKeyChecking=no` 绕过。

出现 `Permission denied (publickey)` 时，主机身份阶段通常已经通过。按顺序检查有效配置、agent 和实际连接：

```bash
ssh -G git@github.com |
  grep -E '^(hostname|user|port|identityfile|identitiesonly) '
ssh-add -l -E sha256
ssh -vT git@github.com
```

如果要隔离验证本篇专用密钥：

```bash
KEY_PATH="$HOME/.ssh/id_ed25519_github"

ssh -vT -o IdentitiesOnly=yes -i "$KEY_PATH" git@github.com
```

确认以下事项：

1. GitHub SSH URL 使用远程用户 `git`，而不是 GitHub 用户名。
2. `identityfile` 或 agent 指纹对应预期私钥。
3. 对应公钥已登记到正确平台账号。
4. 账号没有被组织策略、SSO 或密钥有效期限制。
5. 通过 `ssh -T` 映射到的账号是预期账号。

不要使用 `sudo git push` 或 `sudo ssh`；它们会切换到 root 的 HOME、SSH 配置和密钥。`-v`/`-vvv` 输出可能包含用户名、地址、配置路径和公钥指纹，分享前必须脱敏。命令选择边界见 [[OpenSSH 常用命令基础#2.5 高频选项按问题记忆]]。

连接超时或被拒绝发生在用户认证之前，应检查实际主机、端口、DNS、网络和组织出口策略。Git 的 HTTP proxy 配置不会自动修复 SSH 路径。

### 7.3 HTTPS：先网络与 TLS，再检查 401/403

证书或代理错误发生在平台验证账号之前。先读取 Git 自身的 HTTP/远程代理配置：

```bash
git config --show-origin --get-regexp \
  '^(http\..*proxy|remote\..*\.proxy)$'
```

没有输出通常表示当前可见 Git 配置没有匹配项；环境变量、系统代理、VPN、DNS 和企业根证书仍可能影响连接。出站代理是 Git 客户端先连接代理服务，再由代理访问代码平台；HTTPS 与 SSH 使用不同协议和配置入口。

公司网络应使用组织提供的代理、根证书和代码平台地址。不要设置 `http.sslVerify=false`，它会关闭身份校验并掩盖中间人风险。如果需要安全查看代理环境变量、比较直连与代理、限定配置范围并回退，统一使用 [[Linux 开发环境出站代理配置与分层排查#9. Git：先区分 HTTPS 远程和 SSH 远程]]。

TLS 和网络路径正常后，HTTPS 返回 401、403 或反复要求认证时，再检查：

```bash
git remote -v
git config --show-origin --get-all credential.helper
```

- 401 通常表示凭据无效、过期或不适用于当前主机。
- 403 可能表示认证成功但账号没有仓库权限，也可能涉及令牌作用域或组织 SSO 授权。
- 反复提示可能是没有可用 helper、helper 无法写入安全存储，或旧条目仍被优先读取。

不要反复输入网页登录密码。根据平台当前要求检查 OAuth、令牌类型、最小作用域、有效期和组织授权，再通过对应安全存储删除旧条目并重新认证。

### 7.4 账号认证成功但仓库仍无法访问

SSH 测试显示预期账号，或 HTTPS 登录已经完成，只能证明平台识别了账号。继续检查：

1. 远程 URL 中的组织、账号和仓库名是否准确。
2. 仓库是否已经重命名、迁移、归档或删除。
3. 当前账号是否拥有仓库读取权限。
4. 推送时，目标分支是否受保护，令牌是否具有写入作用域，组织是否要求额外 SSO 授权。

先用 `git ls-remote` 验证读取。写入权限留到受控的功能分支推送中验证，不要为了测试权限向 `main` 写入无意义提交。

## 8. 完成检查与下一步

离开本篇前确认：

- [ ] 已完成第一次本地提交，并能区分提交身份与远程认证。
- [ ] 已解释本地仓库、远程 URL、`origin`、网络、凭据和仓库权限的关系。
- [ ] 已选择 HTTPS 或 SSH 中的一条路径，没有同时盲目修改两套配置。
- [ ] HTTPS 凭据由受认可的 helper 管理，或 SSH 私钥只保留在可信客户端。
- [ ] SSH 场景已区分私钥文件、`~/.ssh/config`、agent 进程与口令存储；自定义密钥路径已有可验证的日常选择规则。
- [ ] 已明确 `AddKeysToAgent` 只能复用正在运行的 agent，没有把 `eval "$(ssh-agent -s)"` 无条件写入 Shell 启动文件。
- [ ] SSH 场景已核对主机指纹、平台账号和实际使用的密钥。
- [ ] `git remote -v` 不含令牌，`git ls-remote` 已验证目标仓库可读取。
- [ ] 已明确：读取验证不等于拥有推送权限。

如需设置 `fetch.prune`、`pull.ff` 等保守默认值，再阅读 [[Git 常用配置与本地验证#9. 按需设置保守的远程协作默认值]]。Pull Request（PR，拉取请求）是把分支改动提交给团队检查、评审和合并的协作入口；第一次受控推送与 PR 流程见 [[Git 分支与 PR 工作流]]，提交标题和正文规范见 [[Git 提交消息编写规范]]。

## 官方参考资料

- [Git：gitcredentials 手册](https://git-scm.com/docs/gitcredentials)
- [Git：凭据帮助程序](https://git-scm.com/doc/credential-helpers)
- [Git：git-credential-store 手册](https://git-scm.com/docs/git-credential-store)
- [Git：git-clone 手册](https://git-scm.com/docs/git-clone)
- [Git：git-remote 手册](https://git-scm.com/docs/git-remote)
- [Git：git-ls-remote 手册](https://git-scm.com/docs/git-ls-remote)
- [OpenBSD：`ssh-agent(1)` 手册](https://man.openbsd.org/ssh-agent.1)
- [OpenBSD：`ssh_config(5)` 手册](https://man.openbsd.org/ssh_config.5)
- [GitHub：远程仓库与 HTTPS/SSH URL](https://docs.github.com/en/get-started/git-basics/about-remote-repositories)
- [GitHub：使用 SSH 连接](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- [GitHub：生成 SSH 密钥并加入 ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [GitHub：测试 SSH 连接](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)
- [GitHub：SSH 主机指纹](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints)
- [GitHub：排查 Permission denied (publickey)](https://docs.github.com/en/authentication/troubleshooting-ssh/error-permission-denied-publickey)
