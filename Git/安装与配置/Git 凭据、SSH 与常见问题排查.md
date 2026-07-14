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
updated: 2026-07-14T22:52:26
---

本文说明 Git 已经安装且本地身份已配置后，如何安全访问远程仓库，并定位常见的 SSH、HTTPS、凭据与网络问题。它以 GitHub 为 SSH 示例平台，但“检查远程地址、保护私钥、使用系统凭据存储、验证真实访问权限”的原则同样适用于 GitLab、Gitea 和公司自建平台。

先完成 [[Git 常用配置与本地验证]]。如果当前问题是 Git 命令本身不存在或版本不符合要求，请先回到 [[Ubuntu 从零安装 Git]] 或 [[macOS 从零安装 Git]]。

> [!important] 远程认证与提交身份是两回事
> `user.name` 和 `user.email` 决定提交历史显示的作者信息；SSH 密钥、访问令牌和平台权限决定你能否读取或推送远程仓库。前者正确并不代表后者已配置成功。

## 先选择 HTTPS 还是 SSH

| 协议 | 远程地址示例 | 认证方式 | 更适合什么情况 |
| --- | --- | --- | --- |
| HTTPS | `https://github.com/ACCOUNT/REPOSITORY.git` | 浏览器 OAuth、个人访问令牌或系统凭据帮助程序 | 网络限制 SSH、首次接触平台、组织统一使用 HTTPS |
| SSH | `git@github.com:ACCOUNT/REPOSITORY.git` | 本机私钥、ssh-agent 和平台登记的公钥 | 经常操作远程仓库、希望避免重复输入令牌、已能管理密钥 |

两种协议都能访问同一个仓库，也不改变提交内容。选择应服从团队、组织安全要求和网络环境；不要为了“看起来高级”切换协议。无论选择哪一种，都不要把密码、访问令牌或私钥提交到仓库。

## 1. 先检查当前仓库与凭据设置

在已经 clone 的仓库根目录中执行：

```bash
git remote -v
git remote get-url origin
git config --show-origin --get-all credential.helper
git help -a | grep credential- || true
```

`git remote -v` 显示 fetch 与 push 使用的地址，`git remote get-url origin` 更适合确认一个远程的实际 URL。没有 `origin` 并不是错误：本地 `git init` 后尚未添加远程仓库时，本来就没有远程认证需求。

不要把含有访问令牌的 URL 复制到笔记、终端历史截图或 Issue。正常的远程 URL 不应包含密码或令牌。

## 2. SSH：创建并使用专用密钥

### 2.1 先检查已有密钥，避免覆盖

```bash
ls -al ~/.ssh 2>/dev/null || true
ssh-add -l 2>/dev/null || true
```

常见的公钥文件名有 `id_ed25519.pub`、`id_ecdsa.pub` 和 `id_rsa.pub`。有现成密钥不代表它已登记到当前代码平台，也不代表 ssh-agent 正在使用它；先确认用途，再决定复用或新建。

私钥没有 `.pub` 后缀，绝不能上传、发送、粘贴或提交。公钥文件才是交给代码平台的内容。

### 2.2 生成 Ed25519 密钥

如果没有合适密钥，或希望将工作与个人用途分开，可生成新的 Ed25519 密钥：

```bash
ssh-keygen -t ed25519 -C "you@example.com"
```

`-C` 后的邮箱只是密钥注释，用于帮助你识别密钥；它不等于 Git 提交身份，也不是平台登录凭据。命令询问保存位置时，已有同名密钥就不要直接覆盖：可使用清晰的专用名称，例如 `~/.ssh/id_ed25519_github_work`。

建议为私钥设置强口令。即使设备磁盘被访问，口令也能为私钥增加一层保护；后续由 ssh-agent 或系统钥匙串减少重复输入。

若使用默认文件名，常见权限可检查为：

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

若你使用了自定义文件名，请将命令中的路径替换为实际文件。不要对整个 HOME 目录递归执行 `chmod`，也不要用 `sudo` 生成用户自己的 SSH 密钥。

### 2.3 将密钥加入 ssh-agent

Ubuntu 和其他常见 Linux 桌面环境可以先启动 agent，再添加私钥：

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l
```

macOS 可使用系统自带的 `ssh-add`，并将口令保存在钥匙串中：

```bash
eval "$(ssh-agent -s)"
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
ssh-add -l
```

`ssh-add -l` 会列出已加载密钥的指纹，不会显示私钥内容。macOS 的 `--apple-use-keychain` 仅适用于苹果系统自带 OpenSSH；不要把该参数复制到 Ubuntu。密钥较多、使用自定义文件名或需要自动加载时，可按平台 SSH 文档在 `~/.ssh/config` 中明确设置 `Host`、`IdentityFile` 与 agent 行为；不要把私钥路径或密钥本体写进 Git 仓库。

### 2.4 将公钥登记到平台

先显示**公钥**并复制其单行内容：

```bash
cat ~/.ssh/id_ed25519.pub
```

然后在代码平台的账号安全设置中新增 SSH key。GitHub 的入口和完整步骤见 [GitHub：生成 SSH 密钥并添加到 ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)。GitLab、Gitea 或公司平台有同类入口，应以对应平台的当前文档为准。

### 2.5 测试 SSH 连接并设置远程 URL

以 GitHub 为例，公钥已登记后运行：

```bash
ssh -T git@github.com
```

首次连接时，先核验终端显示的主机指纹与 [GitHub 官方 SSH 指纹](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints) 一致，再接受。GitHub 成功认证后会显示账号名和“不提供 shell access”之类的信息；该测试命令返回非零退出码在 GitHub 场景也可能是正常结果，应以消息内容和官方说明判断。

把已有仓库的远程改为 SSH URL 前，先记录原地址：

```bash
git remote -v
git remote set-url origin git@github.com:ACCOUNT/REPOSITORY.git
git remote -v
git ls-remote origin HEAD
```

`git ls-remote origin HEAD` 只读取远程引用，适合在不改动工作区的情况下验证访问。将 `ACCOUNT/REPOSITORY` 替换为真实路径；不要把示例地址原样复制到自己的仓库。

## 3. HTTPS：使用平台令牌与系统凭据存储

HTTPS 是完全正常的 Git 访问方式。许多平台不再接受账户密码用于 Git 操作，而是使用个人访问令牌、OAuth 登录或平台提供的专用凭据流程。令牌权限和有效期应遵循平台、组织和仓库的最小权限要求。

Git 通过 credential helper 与操作系统凭据存储交互。先查看当前可用的帮助程序：

```bash
git help -a | grep credential- || true
git config --show-origin --get-all credential.helper
```

### macOS：使用 Keychain

若当前 Git 提供 `osxkeychain` 帮助程序，可显式启用它：

```bash
git config --global credential.helper osxkeychain
git config --global --get credential.helper
```

之后首次访问需要认证的 HTTPS 远程时，按平台提示完成 OAuth 或输入令牌；Git 会通过 Keychain 保存凭据。若系统中没有这个帮助程序，不要强行添加配置；先确认 Git 的安装来源及该帮助程序是否随其提供。

### Ubuntu：优先使用 Secret Service 集成

Linux 上常见的安全持久化帮助程序是 `libsecret`，它可与 GNOME Keyring、KDE Wallet 等系统密钥服务集成。只有当 `git help -a` 的输出确实列出了 `credential-libsecret` 时，才配置：

```bash
git config --global credential.helper libsecret
git config --global --get credential.helper
```

不同 Ubuntu 版本、桌面环境和 Git 打包方式可能让该帮助程序由额外的软件包提供。若它不存在，请使用组织批准的 Git Credential Manager、OAuth helper 或 SSH；不要因为图省事改用明文存储。

### 仅临时缓存的选择

在安全持久化存储不可用、且确实只需要短时认证时，可使用内存缓存：

```bash
git config --global credential.helper "cache --timeout=3600"
```

该设置让凭据在内存中缓存约一小时；到期或系统重启后需要重新认证。它仍应仅用于你理解并接受的个人环境，不应替代组织要求的凭据方案。

> [!danger] 不要使用明文 `store` 保存长期令牌
> `git config --global credential.helper store` 会把密码或令牌以未加密形式写入磁盘。Git 官方文档明确说明这只受文件权限保护，不适合作为常规长期方案。优先使用 macOS Keychain、Linux Secret Service、受认可的凭据管理器或 SSH 密钥。

## 4. 更换、撤销或清理凭据

凭据问题不能只靠 `git config --unset` 解决：该命令最多删除“使用哪个 helper”的配置，不会自动删除 Keychain、Secret Service 或平台中已经保存的令牌。

正确顺序通常是：

1. 在代码平台撤销泄露、过期或不再需要的访问令牌，或删除不再使用的 SSH 公钥。
2. 在 macOS Keychain、Linux 的密钥服务或组织指定凭据管理器中删除旧条目。
3. 通过 `git config --show-origin --get-all credential.helper` 检查 Git 仍会调用哪个 helper。
4. 必要时再修改或移除 helper 配置，例如 `git config --global --unset-all credential.helper`。
5. 使用 `git ls-remote origin HEAD` 或一次受控的 fetch 验证新凭据。

若怀疑令牌已经暴露，应先撤销令牌，再排查日志、截图、Shell 历史、仓库远程 URL 和配置文件。不要把“改了一个令牌”当作完整的泄露处置。

## 5. 常见错误与排查顺序

### `Permission denied (publickey)`

按以下顺序检查：

```bash
git remote get-url origin
ssh-add -l
ssh -vT git@github.com
```

确认远程是 `git@github.com:...` 形式，agent 中确实有预期密钥，且调试输出显示尝试的是该密钥。GitHub SSH URL 应使用用户 `git`，而不是你的 GitHub 用户名；也不要使用 `sudo git push`，否则会切换到 root 的密钥和配置。

如果平台拒绝密钥，检查公钥是否已登记到正确账号、该账号是否有仓库权限、以及是否误用了另一套工作/个人密钥。GitHub 的具体错误说明见 [GitHub：排查 Permission denied (publickey)](https://docs.github.com/en/authentication/troubleshooting-ssh/error-permission-denied-publickey)。

### HTTPS 返回 401、403 或反复要求认证

先确认远程地址和平台账号：

```bash
git remote -v
git config --show-origin --get-all credential.helper
```

401 通常表示凭据无效、过期或不适用于该主机；403 也可能表示认证成功但账号没有仓库权限。不要把网页登录密码反复粘贴到终端。检查平台要求的令牌类型、作用域、有效期和组织 SSO 授权，然后在系统凭据存储中删除旧条目后重新认证。

### 首次 SSH 连接提示主机真实性无法建立

这是 SSH 首次连接的正常保护提示，但不能不看内容就输入 `yes`。先与平台官方公布的 SSH 主机指纹核对；若指纹不匹配或网络环境异常，立即停止。不要删除 `~/.ssh/known_hosts` 来绕过提示。

### HTTPS 出现证书或代理错误

先检查是否存在历史代理或 Git HTTP 配置：

```bash
git config --show-origin --get-regexp '^(http|https)\.' || true
env | grep -iE '^(http|https|no)_proxy=' || true
```

公司网络应使用组织提供的代理、根证书和代码平台地址。不要用 `http.sslVerify=false` 关闭 TLS 校验；这会掩盖证书与中间人风险，而不是修复网络问题。

## 6. 认证成功后的日常工作

认证只解决“能访问远程仓库”。创建分支、审查暂存内容、提交和发起 PR 仍应遵循 [[Git 分支与 PR 工作流]]；提交说明见 [[Git 提交消息编写规范]]。当前工作未完成却需要验证其他分支时，优先使用 [[Git 未完成开发时安全验证其他分支代码]]，不要为测试目的把别人的分支直接拉进当前分支。

## 官方参考资料

- [Git：凭据帮助程序](https://git-scm.com/doc/credential-helpers)
- [Git：gitcredentials 手册](https://git-scm.com/docs/gitcredentials)
- [Git：git-credential-store 手册](https://git-scm.com/docs/git-credential-store)
- [GitHub：使用 SSH 连接](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
- [GitHub：测试 SSH 连接](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/testing-your-ssh-connection)
