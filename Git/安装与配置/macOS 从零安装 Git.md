---
title: macOS 从零安装 Git
aliases:
  - Mac 安装 Git
  - macOS 安装 Git
  - macOS Git 配置前准备
tags:
  - Git
  - Git/安装
  - Git/macOS
  - Homebrew
  - Xcode
created: 2026-07-14T22:52:26
updated: 2026-07-14T22:52:26
---

本文面向使用 macOS 的个人开发者，说明如何识别系统现有 Git，并通过 Xcode Command Line Tools 或 Homebrew 安装、更新和验证 Git。两种方式都可以使用；重点不在于“装得最多”，而在于知道当前终端究竟运行哪个版本。

先阅读 [[Git 安装与初始配置概览]] 了解整体路径。安装完成后继续 [[Git 常用配置与本地验证]]；要访问 GitHub、GitLab 或公司的私有仓库时，再阅读 [[Git 凭据、SSH 与常见问题排查]]。

## 目标与安装方式选择

| 你的情况 | 推荐方式 | 原因 |
| --- | --- | --- |
| 只需基本 Git，且希望使用苹果系统工具链 | Xcode Command Line Tools | 系统集成度高，安装步骤少 |
| 已使用 Homebrew 管理开发工具 | Homebrew | 更新与其他命令行工具统一管理 |
| 项目明确要求比系统工具链更新的 Git | Homebrew 或团队指定的受管方案 | 可明确控制并验证版本 |
| 公司设备限制安装软件 | 先咨询管理员 | 不能以绕过策略为目标安装工具 |

无论选择哪种方式，最后都要用 `command -v git`、`type -a git` 和 `git --version` 验证结果。系统自带 Git 与 Homebrew Git 可以共存，PATH 的顺序决定实际使用哪一个。

## 1. 先记录当前状态

以下命令均不会改动系统：

```bash
command -v git || true
type -a git 2>/dev/null || true
git --version 2>/dev/null || true
xcode-select -p 2>/dev/null || true
command -v brew || true
```

`command -v git` 显示当前 Shell 优先使用的路径；`type -a git` 会显示所有可找到的 Git。若已经能运行 Git，先记录版本，再决定是否确实需要另装一个版本。

`xcode-select -p` 若能输出开发者工具目录，通常表示 Command Line Tools 或完整 Xcode 已处于已选中状态；它不代表 Homebrew Git 已安装。`command -v brew` 有输出则表示当前 Shell 已能找到 Homebrew。

> [!tip] 不要只看 Finder 里是否安装了 Xcode
> 完整 Xcode、Command Line Tools、系统自带 Git、Homebrew Git 是不同层次的东西。对 Git 而言，终端里的实际路径与版本才是最终事实。

## 2. 路线 A：安装 Xcode Command Line Tools

这是 macOS 上最直接的基础安装路径。可以先执行：

```bash
git --version
```

若系统尚未安装命令行开发工具，macOS 通常会弹出安装提示。也可以显式运行：

```bash
xcode-select --install
```

按照图形界面完成安装后，关闭并重新打开终端，再验证：

```bash
command -v git
git --version
xcode-select -p
```

Git 官方安装说明将 Xcode Command Line Tools 列为 macOS 上的简便安装方式。该工具集还包含编译器和其他开发基础组件，因此安装时间和占用空间通常比单个 Git 包更大。

### 如何更新这一路线

Command Line Tools 通常随 macOS 的软件更新或开发工具更新维护。若当前版本已满足项目要求，不需要为 Git 单独频繁更新。项目明确要求更高版本时，优先考虑下一节的 Homebrew 路线，或采用团队统一的受管工具链。

不要手工删除 `/usr/bin/git`，也不要以 `sudo` 运行日常 Git 命令。系统工具由系统维护，手工删除会造成不可预测的开发工具问题。

## 3. 路线 B：使用 Homebrew 安装 Git

若你已经使用 Homebrew，或确实需要由包管理器维护较新的 Git，可使用这一路线。若还没有 Homebrew，请先阅读 [Homebrew 官方安装说明](https://brew.sh/) 并确认它适合当前设备与组织策略；不要从搜索结果或不明镜像复制安装脚本。

先检查 Homebrew 是否可用：

```bash
brew --version
brew doctor
```

然后更新 Homebrew 的索引并安装 Git：

```bash
brew update
brew install git
```

安装后检查 Homebrew 所管理的 Git 与当前实际生效的 Git：

```bash
brew info git
brew --prefix git
command -v git
type -a git
git --version
git --exec-path
```

Apple Silicon 的 Homebrew 默认前缀通常是 `/opt/homebrew`，Intel Mac 通常是 `/usr/local`；但不要把这两个路径硬编码到个人配置中，应以 `brew --prefix` 和 `command -v git` 的输出为准。

### 安装后仍然显示旧版本怎么办

新开一个终端，然后重新执行：

```bash
command -v git
type -a git
git --version
```

若优先路径仍是 `/usr/bin/git`，说明 Homebrew 的 shell 环境尚未位于 PATH 前面。应按照 Homebrew 安装完成时显示的官方 `shellenv` 提示处理，而不是猜测并向多个启动文件重复写 PATH。若你使用 Zsh，先确认将 PATH 设置放在正确的启动文件；相关 Shell 启动文件的区别可参考 [[macOS 从零配置 Oh My Zsh]]。

> [!warning] 不要为了切换 Git 版本删除系统 Git
> 多个 Git 版本共存并不危险，难点在于你是否知道当前用了哪一个。保留系统 Git，通过 PATH 和验证命令选择实际版本，比删除系统文件安全得多。

## 4. 更新或卸载 Homebrew Git

Homebrew 管理的 Git 可以单独更新：

```bash
brew update
brew upgrade git
git --version
```

确实不再需要 Homebrew Git 时，可以卸载该 formula：

```bash
brew uninstall git
```

卸载后再次执行 `command -v git` 与 `git --version`。如果 Command Line Tools 仍存在，终端通常会回到系统 Git；如果没有任何 Git，按路线 A 或项目指定的方式重新安装即可。

卸载 Homebrew Git 不会删除你的仓库、`~/.gitconfig`、SSH 密钥或平台访问令牌。它只删除 Homebrew 管理的 Git 程序及其关联文件。

## 5. 常见问题

### 每次运行 Git 都要求安装 Command Line Tools

先完成图形安装并等待系统结束，再新开终端验证。如果安装窗口反复出现，可能是安装未完成、当前开发者目录异常，或设备受管理策略限制。保留弹窗与 `xcode-select -p` 的输出，向管理员或 Apple 支持渠道确认；不要通过复制未知目录来伪造 Command Line Tools。

### `brew install git` 成功，但 `git --version` 没有变化

用 `type -a git` 确认是否仍优先使用 `/usr/bin/git`。随后检查当前 Shell 是否已加载 Homebrew 的 PATH 设置。不要在 `~/.zshrc`、`~/.zprofile`、`~/.zshenv` 中同时添加多份相互矛盾的 PATH；每次调整后新开终端并验证。

### IDE 与终端显示不同的 Git 版本

IDE 可能继承不同的 PATH，或在设置中指定了独立 Git 可执行文件。先在 IDE 的 Git 设置中查看路径，再与终端的 `command -v git` 对比。若团队没有版本要求，优先让两者使用同一来源，减少行为差异。

## 6. 继续配置 Git 本身

安装成功后，Git 还不知道提交应该署谁的名字和邮箱。请继续 [[Git 常用配置与本地验证]]；完成后可按照 [[Git 分支与 PR 工作流]] 开始日常分支开发。需要设置 SSH 或 HTTPS 凭据时，进入 [[Git 凭据、SSH 与常见问题排查]]。

## 官方参考资料

- [Git：在 macOS 上安装 Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
- [Apple：安装 Xcode Command Line Tools](https://developer.apple.com/library/archive/technotes/tn2339/_index.html)
- [Homebrew 官方网站](https://brew.sh/)
- [Git：git-config 手册](https://git-scm.com/docs/git-config)
