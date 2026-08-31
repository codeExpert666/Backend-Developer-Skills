---
title: 使用 Git 与 GNU Stow 搭建 dotfiles 仓库
aliases:
  - Dotfiles 仓库搭建与跨机器部署
  - GNU Stow 配置管理
  - 现代终端配置仓库
tags:
  - Terminal
  - Terminal/使用
  - Git
  - Dotfiles
  - GNU-Stow
created: 2026-08-28T16:05:55
updated: 2026-08-31T13:18:23
---

这篇专题回答 dotfiles 的稳定问题：配置源在哪里、Stow 把它部署到哪里、Git 跟踪什么、冲突如何处理、秘密与运行数据为什么必须留在仓库之外。

> [!tip] 何时打开本文
> 从零搭建或恢复新机器时直接执行对应主线。只有在设计新软件包、接管已有文件、理解链接归属、固化部署脚本或处理 Git 生命周期时，才需要打开本文。

平台软件安装与完整执行顺序分别见 [[macOS 从零搭建现代终端环境]]、[[Ubuntu 从零搭建现代终端环境]]、[[从 Oh My Zsh 迁移到 Antidote]] 和 [[从已有 dotfiles 恢复现代终端环境]]；本文不再维护第五套从零流程。

> [!info] 资料核对范围
> 本文于 2026-08-28 核对 GNU Stow 2.4.1、Git、XDG Base Directory、chezmoi 与 yadm 的官方资料。资料核对不等于某台机器已经部署成功。

## 1. 先分清源、目标、状态与秘密

| 层次 | 示例 | 谁负责 | 丢失后如何恢复 |
| --- | --- | --- | --- |
| 配置源 | 仓库中的 `.zshrc`、`starship.toml` | Git | 从提交或远端恢复 |
| 生效目标 | `$HOME/.zshenv`、`$HOME/.config/zsh/.zshrc` | Stow 创建的符号链接 | 从配置源重新部署 |
| 本机覆盖 | `local.zsh`、`local.zprofile` | 当前机器的真实文件 | 私密备份或重新填写 |
| 运行数据 | 历史、Atuin/zoxide 数据库、缓存 | 各应用 | 应用同步、私密备份或重新生成 |
| 秘密 | 令牌、私钥、密码 | 密钥链或秘密管理器 | 按秘密系统恢复与轮换 |

关系可以压缩为：

```text
普通 Git 仓库中的配置源
          │
          │ GNU Stow 创建或维护链接
          ▼
$HOME 下程序真正读取的目标
          │
          ├── 同目录中的本机真实 local 文件
          └── XDG data/state/cache 中的运行数据
```

“目标位于 `$HOME/.config`”由 XDG 或应用约定决定；“目标链接到仓库的哪份源”由 Stow 决定；“源是否形成版本历史”由 Git 决定。三者不能互相替代，路径职责见 [[XDG 基础目录与终端配置边界]]。

### 1.1 先决定谁拥有第一次写入

Stow 只能为不存在的目标创建链接，或维护已经属于正确软件包的链接。若应用先在同一路径生成普通文件，Stow 会报告冲突；这不是 Stow 失效，而是两个所有者同时争用同一个目标。

因此，在第一次执行会加载完整配置的命令前，先列出“启动依赖闭包”：

| `.zshrc` 或终端会启动的组件 | 准备由 Stow 管理的配置 | 首次执行前必须做什么 |
| --- | --- | --- |
| Zsh | `.zshenv`、`.zprofile`、`.zshrc`、插件清单 | 部署 `zsh` 并验证链接 |
| Atuin | `~/.config/atuin/config.toml` | 先部署 `atuin`，再运行 `atuin info`、`atuin doctor` 或 `atuin init` |
| Starship | `~/.config/starship.toml` | 先部署 `starship`，再运行完整交互 Zsh 或 `starship explain` |
| Ghostty | `~/.config/ghostty/config.ghostty` | 先部署 `ghostty`，再首次启动应用 |
| Antidote | 插件清单由 `zsh` 管理；clone 与静态加载文件不受管 | 把 Antidote 本体放入 data，插件 clone 与生成文件放入 cache |
| zoxide、fzf | 本方案没有独立受管配置 | 数据库、Shell 生成代码与缓存留在仓库外 |

`zsh -n file` 只检查语法，不执行文件内容，可以在部署前使用。`zsh -ic`、`zsh -lic` 和打开新的终端窗口会真正运行初始化，必须放在完整软件包列表部署之后。只设置临时 `ZDOTDIR` 不是完整隔离：Atuin、Starship 等仍会看到真实 `$HOME` 与 XDG 目录。

## 2. Stow 软件包如何映射到 `$HOME`

本文把 dotfiles 仓库根目录下、准备作为一个整体部署的目录称为 Stow package（软件包）。软件包内部镜像目标目录相对于 `$HOME` 的结构：

```text
$DOTFILES_DIR/
├── README.md
├── scripts/
├── zsh/
│   ├── .zshenv
│   └── .config/zsh/
│       ├── .zprofile
│       ├── .zshrc
│       └── .zsh_plugins.txt
├── starship/
│   └── .config/starship.toml
├── atuin/
│   └── .config/atuin/config.toml
└── ghostty/
    └── .config/ghostty/config.ghostty
```

因此：

| 仓库源 | 部署目标 |
| --- | --- |
| `zsh/.zshenv` | `$HOME/.zshenv` |
| `zsh/.config/zsh/.zshrc` | `$HOME/.config/zsh/.zshrc` |
| `starship/.config/starship.toml` | `$HOME/.config/starship.toml` |
| `atuin/.config/atuin/config.toml` | `$HOME/.config/atuin/config.toml` |
| `ghostty/.config/ghostty/config.ghostty` | `$HOME/.config/ghostty/config.ghostty` |

`README.md` 和 `scripts/` 是仓库说明与运维入口，不是软件包，不能传给 Stow。

本文始终使用 `--no-folding`。如果让 Stow 把整个 `$HOME/.config/zsh` 折叠为指向仓库目录的单个链接，随后创建的 `local.zsh` 和生成文件就会实际落入仓库树，源、目标和本机状态的边界会变得模糊。

## 3. 先按现状选择动作

```sh
DOTFILES_DIR="$HOME/.dotfiles"

ls -ld "$DOTFILES_DIR" 2>/dev/null || true
git -C "$DOTFILES_DIR" status --short --branch 2>/dev/null || true
git -C "$DOTFILES_DIR" rev-parse --show-toplevel 2>/dev/null || true
git -C "$DOTFILES_DIR" remote -v 2>/dev/null || true
```

| 现场 | 正确动作 |
| --- | --- |
| 目录不存在，且第一次建立 | 走当前平台“从零”主线，由主线初始化仓库和第一个软件包 |
| 目录不存在，但远端已有可信仓库 | 走 [[从已有 dotfiles 恢复现代终端环境]]，不要 `git init` |
| 目录存在且是自己明确维护的仓库 | 先保留已有修改，再进行本文的软件包或生命周期操作 |
| 目录存在但来源不明、不是 Git 或远端异常 | 停止，查清所有者和来源；不删除、不覆盖、不重新初始化 |

`git init` 返回成功只说明工作区建立，并不提供可回退版本；至少一个经过真实 Shell 或组件验收的提交才是已知良好基线。

## 4. 哪些内容进入仓库

一个判断顺序比“仓库是私有的”更可靠：

1. 它是希望在多台机器复现的稳定声明吗？
2. 它是否含机器路径、账号、内部地址或秘密？
3. 它会不会在每次运行时持续变化？
4. 它能否由安装器、插件清单或程序自动重新生成？

| 内容 | 默认决定 | 原因 |
| --- | --- | --- |
| Zsh 启动文件与插件清单 | 跟踪 | 稳定声明 |
| Starship、Ghostty、Atuin 的非秘密配置 | 只要主线声明由 Stow 管理就跟踪 | 可复现行为，并避免应用抢先生成普通目标 |
| `local.zsh`、`local.zprofile` | 不跟踪 | 本机与账号差异 |
| Shell 历史、Atuin 数据库与密钥 | 不跟踪 | 私人状态或秘密 |
| zoxide 数据库 | 不跟踪 | 持续变化的本机状态 |
| Antidote 本体、插件 clone、生成文件 | 不跟踪 | 可由清单重新生成 |
| 缓存、补全转储、日志 | 不跟踪 | 可重新生成且制造噪声 |
| 原始迁移备份 | 不跟踪 | 可能包含秘密，只用于本机回退 |

私有 Git 远端降低了公开暴露概率，却不会消除误分享、协作者权限、历史永久保留或账号泄露风险。秘密仍应使用钥匙串或秘密管理器。

## 5. 安全接管已经存在的目标

Stow 遇到与源路径同名的普通文件通常会报告冲突。冲突是保护信号，不是要绕过的错误。

以 `.zshenv` 为例，先检查三方：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
source_path="$DOTFILES_DIR/zsh/.zshenv"
target_path="$HOME/.zshenv"

ls -ld "$source_path" "$target_path" 2>/dev/null || true
readlink "$target_path" 2>/dev/null || true
git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" ls-files -- zsh/.zshenv
```

若目标是普通文件，按以下顺序处理：

1. 确认有一份权限收紧、位于仓库外的备份；
2. 用 `diff -u` 比较当前目标和准备部署的源；
3. 人工决定哪些行为进入共享源、哪些进入 local、哪些删除；
4. 对源做语法或领域检查；
5. 只把这个明确目标移动到备份目录；
6. 再次运行 Stow 模拟。

```sh
DOTFILES_DIR="$HOME/.dotfiles"
diff -u "$HOME/.zshenv" "$DOTFILES_DIR/zsh/.zshenv" || true
```

不要使用 `stow --adopt` 自动把未知目标移进仓库：它会改变源树，容易把本机私有值或旧配置直接变成待提交内容。也不要为了一个文件冲突而移动整个 `$HOME/.config`。

## 6. 部署事务：同一列表先模拟，再应用

只部署真实存在、当前机器安装了对应程序，并且属于本次启动依赖闭包的软件包。本套命令行基线使用 `zsh atuin starship`：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --restow --verbose=2 zsh atuin starship
```

逐行检查模拟输出：

- 软件包名是否正确；
- 目标是否都位于当前 `$HOME`；
- 是否出现 conflict；
- 是否会删除不属于本次软件包的链接；
- 是否误把 `scripts` 或仓库根文件当成软件包。

确认后，只移除 `--simulate`，其余参数和软件包列表原样保持：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --restow --verbose=2 zsh atuin starship
```

macOS 桌面主线还会把 `ghostty` 追加到这两个命令。加入或移除软件包时，模拟和应用必须使用完全相同的显式列表。不要使用“部署仓库中所有目录”的通配方式；README、scripts、未来目录和当前机器不适用的软件包不应被猜测部署。

## 7. 可选：固化统一部署入口

第一次手工部署成功后，再把参数固化为 `$DOTFILES_DIR/scripts/deploy`：

```sh
#!/bin/sh
set -eu

script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd -P)
repo_dir=$(CDPATH= cd -- "$script_dir/.." && pwd -P)
target_dir=${HOME:?HOME is required}

if [ "$#" -lt 2 ]; then
  printf 'usage: %s [--simulate|--apply] package...\n' "$0" >&2
  exit 2
fi

mode=$1
shift
case "$mode" in
  --simulate) action=simulate ;;
  --apply) action=apply ;;
  *)
    printf 'usage: %s [--simulate|--apply] package...\n' "$0" >&2
    exit 2
    ;;
esac

for package_name in "$@"; do
  case "$package_name" in
    zsh|starship|atuin|ghostty) ;;
    *)
      printf 'unsupported package: %s\n' "$package_name" >&2
      exit 2
      ;;
  esac

  if [ ! -d "$repo_dir/$package_name" ]; then
    printf 'package does not exist: %s\n' "$repo_dir/$package_name" >&2
    exit 2
  fi
done

if [ "$action" = simulate ]; then
  exec stow --dir="$repo_dir" --target="$target_dir" --no-folding \
    --simulate --restow --verbose=2 "$@"
fi

exec stow --dir="$repo_dir" --target="$target_dir" --no-folding \
  --restow --verbose=2 "$@"
```

然后：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
chmod u+x "$DOTFILES_DIR/scripts/deploy"
"$DOTFILES_DIR/scripts/deploy" --simulate zsh atuin starship
```

脚本不自动安装软件、创建 local 文件或猜 package。新增软件包时，必须同步审查允许列表与 README。

## 8. 验证链接的归属，而不只看“是链接”

`readlink` 能显示链接记录的目标，却不能单独证明它来自预期仓库。最低检查应组合 Git、链接和解析结果：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" rev-parse --show-toplevel
git -C "$DOTFILES_DIR" status --short --branch

for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/atuin/config.toml" \
  "$HOME/.config/starship.toml"; do
  if [ ! -L "$managed_path" ]; then
    printf 'not a managed link: %s\n' "$managed_path" >&2
    continue
  fi
  ls -ld "$managed_path"
  readlink "$managed_path"
done

zsh -n "$HOME/.zshenv"
zsh -n "$HOME/.config/zsh/.zprofile"
zsh -n "$HOME/.config/zsh/.zshrc"
```

语法通过不等于插件存在、交互启动成功或 SSH 路径正确。组件级和真实新会话验收属于场景主线或 [[现代终端环境更新、验证与回退]]。

## 9. Git 提交与远端边界

提交前先审查未暂存内容：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff --check
git -C "$DOTFILES_DIR" diff
```

显式暂存本次软件包和说明，不使用不受控的 `git add .`：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" add -- README.md zsh atuin starship
if [ -d "$DOTFILES_DIR/scripts" ]; then
  git -C "$DOTFILES_DIR" add -- scripts
fi
git -C "$DOTFILES_DIR" diff --cached --check
git -C "$DOTFILES_DIR" diff --cached
```

再搜索常见秘密标记。它只能发现线索，不能替代人工阅读：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
git -C "$DOTFILES_DIR" grep --cached -nEi \
  '(password|passwd|token|secret|private[ _-]?key|api[ _-]?key)' -- . || true
```

只有真实组件和新会话验收通过，才创建已知良好提交。远端是灾难恢复与跨机器分发手段，不是本地 Git 成立的前提；推送后还应核对远端分支确实包含当前提交。

## 10. 日常修改闭环

受管目标是符号链接，因此编辑 `$HOME/.config/zsh/.zshrc` 通常会修改仓库源。每次修改遵循：

1. 用 `readlink` 和 `git status` 确认正在编辑的事实来源；
2. 一次只改变一个能力；
3. 先做语法或领域检查，再开新的终端或 SSH 会话；
4. 新增、移动或删除源路径后，再以相同软件包运行 Stow 模拟和应用；
5. 审查 diff 与秘密边界后提交；
6. 保留上一个已知良好提交，直到观察完成。

创建新软件包时，目录应与第一份有效配置同时出现。不要预建空软件包，也不要为“以后可能用”把未安装组件加入部署脚本。

## 11. 新机器恢复只保留一个事实来源

完整恢复流程只在 [[从已有 dotfiles 恢复现代终端环境]] 维护。本文只保留它依赖的契约：

- clone 前目标仓库目录必须不存在或已经明确是同一仓库；
- 部署前先安装软件，阅读 README 和启动文件，得到完整的启动依赖闭包；
- 冲突先备份、比较，再模拟；
- 所有受管组件配置都部署并验证为链接后，才运行真实 Zsh 或首次打开终端应用；
- local、历史、密钥和数据库由各自机制恢复；
- 恢复通常不应改变 Git 工作树。

这样，dotfiles 专题解释模型，恢复主线解释执行顺序，两者不再各自维护一套容易漂移的完整步骤。

## 12. 回退分成内容与部署两件事

回退某次配置内容时，先保留当前工作树，再查看提交历史：

```sh
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff
git -C "$DOTFILES_DIR" log --oneline --decorate -10
```

不要在有未确认修改时使用破坏性重置。优先对明确文件生成反向修改、提交一个回退提交，或从已知良好提交恢复到新分支中测试。

解除 Stow 只移除链接，不回退 Git 内容，也不卸载软件。先模拟：

```sh
DOTFILES_DIR="$HOME/.dotfiles"
stow --dir="$DOTFILES_DIR" --target="$HOME" --no-folding \
  --simulate --delete --verbose=2 zsh
```

确认只涉及该软件包拥有的链接后，移除 `--simulate`。随后再从明确的私密备份恢复原目标；local 文件和运行数据通常仍会留在原位置，应按回退目标决定是否保留。

## 13. 何时重新评估 Stow

当前方案需要大量主机变量、条件模板、密码管理器取值时，可以评估 [chezmoi](https://www.chezmoi.io/)；希望直接以 `$HOME` 为 Git 工作树并使用更完整的 dotfiles 封装时，可以评估 [yadm](https://yadm.io/)。

不要同时让 Stow、chezmoi 和 yadm 管理同一目标。切换工具前必须写清新的源、目标、秘密与回退模型。

## 官方参考资料

- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)
- [GNU Stow：冲突](https://www.gnu.org/software/stow/manual/html_node/Conflicts.html)
- [Git Documentation](https://git-scm.com/doc)
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- [GitHub：Removing sensitive data from a repository](https://docs.github.com/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [chezmoi Documentation](https://www.chezmoi.io/)
- [yadm Documentation](https://yadm.io/)
