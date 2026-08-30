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
updated: 2026-08-29T21:23:53
---

本文为 [[现代终端环境搭建概览]] 建立真正可交付的 dotfiles（个人配置文件）成果：使用一个普通 Git 仓库保存声明式配置，再由 GNU Stow 把仓库中的文件以符号链接部署到 `$HOME`。完成后，配置源文件、实际生效路径、机器私有覆盖和运行数据各有明确边界，并能在 macOS 与 Ubuntu 之间安全恢复。

本文只管理个人配置，不替代平台软件安装。Git 与 Stow 的安装入口分别见 [[Git 安装与初始配置概览]]、[[macOS 从零搭建现代终端环境]] 和 [[Ubuntu 从零搭建现代终端环境]]；终端组件的配置内容、加载顺序和运行验证分别由 [[Zsh 与 Antidote 跨机器配置管理]]、[[Ghostty 常用配置与 Shell 集成]]、[[Starship 提示符配置]]、[[Atuin 命令历史管理]] 和 [[zoxide 与 fzf 导航和模糊查找]] 负责。本文只先说明每类配置文件为什么出现在仓库、由 Stow 部署到哪里，以及如何安全维护它们的生命周期。

> [!info] 资料核对范围
> 本文于 2026-08-28 核对 GNU Stow 2.4.1 官方手册、Git 官方文档、XDG Base Directory Specification、GitHub 安全文档，以及 chezmoi、yadm 官方说明。具体机器尚未按本文执行时，只能称为“教程已整理”，不能称为“本机已部署”。

## 1. 先理解源、目标、状态和秘密

dotfiles 不是把整个主目录提交到 Git，而是只把能够跨机器复用的配置声明放进仓库。本文把相关内容分成四层：

| 层次 | 示例 | 归属 | 丢失后的恢复方式 |
| --- | --- | --- | --- |
| 配置源 | `.zshrc`、`starship.toml`、Ghostty 公共配置 | dotfiles Git 仓库 | 从本地提交或远端仓库恢复 |
| 生效目标 | `~/.zshenv`、`~/.config/zsh/.zshrc` | 由 Stow 创建的符号链接 | 从配置源重新部署 |
| 本机覆盖 | `local.zsh`、`local.zprofile`、`local.ghostty` | 当前机器的真实文件 | 单独备份或在新机器重新填写 |
| 运行数据与秘密 | Atuin 数据库和密钥、zoxide 数据库、缓存、令牌 | Git 之外的状态或秘密存储 | 私有备份、同步服务或重新生成 |

这里的关键关系是：

~~~text
普通 Git 仓库中的源文件
        │
        │ GNU Stow 创建和维护符号链接
        ▼
$HOME 下应用真正读取的目标路径
        │
        ├── 未被 Stow 管理的 local 文件
        └── XDG state、data、cache 中的运行数据
~~~

`$HOME/.config` 是未设置 `XDG_CONFIG_HOME` 时的规范默认配置目录；支持这套规范并采用该位置的应用会从这里读取配置，仓库则只是配置源。[[现代终端环境搭建概览#先理解 XDG 基础目录规范|XDG 基础目录规范]]把配置、持久状态、可重新生成的缓存和用户数据分开，这也是本文不把 Atuin 历史或 zoxide 数据库放进 dotfiles 的基础。

## 2. 为什么主路线选择普通 Git 仓库与 Stow

GNU Stow 是符号链接树管理器。本文把直接位于 dotfiles 仓库根目录、并作为一个整体交给 Stow 部署的配置目录称为 Stow package（配置包）。每个 package 内部都按目标目录的相对结构组织，Stow 再把其中的文件链接到统一目标 `$HOME`。这套现代终端环境只涉及 macOS、Ubuntu 和少量明确的平台差异，使用 Stow 可以保留三个重要特性：

1. dotfiles 是普通 Git 工作区，可直接使用 `git status`、`git diff`、提交和远端。
2. 仓库中的文件保持真实名称和目录结构，不需要先学习模板文件名编码。
3. 使用 `--simulate` 可先观察，冲突时 Stow 会停止；使用 `--no-folding` 可让目标子目录保持真实目录，只为单个受管文件创建链接。

本文所有部署命令都显式使用 `--no-folding`。否则 Stow 可能把整个 `~/.config/zsh` 折叠为指向仓库目录的链接，随后创建的 `local.zsh` 和生成文件就会实际落进仓库工作树，即使被忽略也模糊了边界。

以下需求出现前，不需要切换工具：

| 新需求 | 可重新评估的工具 | 原因 |
| --- | --- | --- |
| 大量主机变量、条件模板和密码管理器取值 | chezmoi | 提供 source state、模板、ignore 和秘密管理器集成 |
| 希望直接以 `$HOME` 为 Git 工作树，并使用 alternate、加密和 bootstrap | yadm | 对 dotfiles 封装 Git 工作流，不要求额外的链接源目录 |

不要在同一套教程中同时部署 Stow、chezmoi 和 yadm。它们对源文件、目标文件和日常修改的模型不同，混用会让“当前事实来源在哪里”再次变得不清楚。

## 3. 安装前提与只读检查

首先确认 Git、提交身份和 Stow。下列命令不修改文件：

~~~bash
command -v git
git --version
git config --global --get user.name
git config --global --get user.email

command -v stow || true
stow --version 2>/dev/null || true
~~~

Git 身份缺失时先完成 [[Git 常用配置与本地验证]]。Stow 可按平台安装：

~~~bash
# macOS：由 Homebrew 管理
brew install stow

# Ubuntu：由 APT 管理
sudo apt update
sudo apt install stow
~~~

同一台机器只保留一个明确的软件来源。安装后再次运行 `command -v stow` 和 `stow --version`，确认当前 Shell 实际调用的版本。

本文用 `DOTFILES_DIR` 表示仓库路径，示例选择 `$HOME/.dotfiles`。它不是行业强制目录，可以替换成其他用户目录，但确定后应始终通过变量引用，不把某台机器的个人绝对路径写入共享配置：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  printf '目标已经存在，请先检查，不能直接初始化：%s\n' "$DOTFILES_DIR" >&2
else
  printf '目标尚不存在，可以创建：%s\n' "$DOTFILES_DIR"
fi
~~~

若目标已经存在，应先执行只读检查：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

ls -ld "$DOTFILES_DIR"
git -C "$DOTFILES_DIR" status --short --branch 2>/dev/null || true
git -C "$DOTFILES_DIR" remote -v 2>/dev/null || true
~~~

不要为了让示例继续而删除、清空或重新初始化已有目录。

## 4. 初始化普通仓库与 package 目录

只有确认 `DOTFILES_DIR` 不存在时，才创建新仓库：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$DOTFILES_DIR" >&2
  exit 1
fi

mkdir -p "$DOTFILES_DIR"
git -C "$DOTFILES_DIR" init
printf '# dotfiles\n' > "$DOTFILES_DIR/README.md"
~~~

`git init` 只创建本地仓库，不会自动创建 GitHub 仓库、添加远端或上传内容。当前分支名取决于 Git 的 `init.defaultBranch`；直接检查真实结果：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" branch --show-current
git -C "$DOTFILES_DIR" status --short --branch
~~~

前面的命令已经把 `$DOTFILES_DIR` 创建为 dotfiles 仓库根目录。后续按平台和组件专题写入第一份有效配置时，会在这个根目录下分别创建 `zsh/`、`ghostty/`、`starship/` 和 `atuin/`；它们各自构成一个前文定义的 Stow package，目录内部镜像 `$HOME` 下的目标结构。`README.md` 和 `scripts/` 也位于仓库根目录，但分别承担仓库说明和运维入口职责，不是要部署到 `$HOME` 的 package。

不要在初始化时批量预建空目录。随着真实配置和运维入口逐步完成，仓库源可以形成下面的结构：

~~~text
$DOTFILES_DIR/
├── README.md
├── zsh/
│   ├── .zshenv
│   └── .config/zsh/
│       ├── .zprofile
│       ├── .zshrc
│       ├── .zsh_plugins.txt
│       ├── common.zsh
│       ├── macos.zsh
│       ├── linux.zsh
│       └── snapshots/
│           └── antidote.txt
├── ghostty/
│   └── .config/ghostty/
│       ├── config.ghostty
│       ├── common.ghostty
│       ├── macos.ghostty
│       └── linux.ghostty
├── starship/
│   └── .config/starship.toml
├── atuin/
│   └── .config/atuin/config.toml
└── scripts/
    ├── deploy
    └── verify
~~~

这棵树是“最终可能形成的仓库源蓝图”，不是初始化后必须立即出现的文件清单，也不是 Stow 部署后的 `$HOME` 目录树。各文件先按下面的职责理解；配置内容和组件级验证由链接的专题继续展开。

| 仓库源路径 | 简要职责与创建时机 | Stow 部署后的去向或边界 | 详细说明 |
| --- | --- | --- | --- |
| `README.md` | fresh clone（从远端全新克隆得到的工作副本）后的第一入口；初始化时先创建标题，首次提交前再补齐真实支持范围、部署和恢复说明 | 留在仓库根目录，不由 Stow 部署 | 本节下文 |
| `zsh/.zshenv` | 所有 Zsh 都会读取的最小引导；开始建立 Zsh 配置时创建，只设置 `ZDOTDIR` 和非交互命令必须看到的最小 PATH | `~/.zshenv` | [[Zsh 与 Antidote 跨机器配置管理]] |
| `zsh/.config/zsh/.zprofile` | 登录 Shell 的环境入口；建立基础 Zsh 骨架时创建，并按需加载不入库的 `local.zprofile` | `~/.config/zsh/.zprofile` | [[Zsh 与 Antidote 跨机器配置管理]] |
| `zsh/.config/zsh/.zshrc` | 交互式 Zsh 的唯一编排入口，负责补全、插件、按键、Atuin、zoxide、fzf 和 Starship 的加载顺序 | `~/.config/zsh/.zshrc` | [[Zsh 与 Antidote 跨机器配置管理]] |
| `zsh/.config/zsh/.zsh_plugins.txt` | Antidote 的声明式插件清单；确定第一组插件时创建，描述“加载什么以及如何加载”，不是生成的加载文件 | `~/.config/zsh/.zsh_plugins.txt` | [[Zsh 与 Antidote 跨机器配置管理]] |
| `zsh/.config/zsh/` 下的 `common.zsh`、`macos.zsh`、`linux.zsh` | 分别保存跨平台、macOS 和 Linux 配置；出现第一项真实内容时才按需创建，`.zshrc` 会按当前平台选择 | `~/.config/zsh/` 下的同名文件 | [[Zsh 与 Antidote 跨机器配置管理]] |
| `zsh/.config/zsh/snapshots/antidote.txt` | 可选的 Antidote 插件版本快照，用完整提交标识保存经过验证的恢复点；只有真实生成 snapshot 后才创建 | `~/.config/zsh/snapshots/antidote.txt` | [[Zsh 与 Antidote 跨机器配置管理]] |
| `ghostty/.config/ghostty/config.ghostty` | Ghostty 主入口，固定公共、平台和本机配置的加载顺序；只在安装 Ghostty 的桌面机器需要该 package | `~/.config/ghostty/config.ghostty` | [[Ghostty 常用配置与 Shell 集成]] |
| `ghostty/.config/ghostty/` 下的 `common.ghostty`、`macos.ghostty`、`linux.ghostty` | 分别保存公共设置和可跟踪的平台模板；需要共享值或出现平台差异时才创建 | `~/.config/ghostty/` 下的同名文件 | [[Ghostty 常用配置与 Shell 集成]] |
| `starship/.config/starship.toml` | Starship 提示符配置源；开始维护默认值以外的提示符行为时创建 | `~/.config/starship.toml` | [[Starship 提示符配置]] |
| `atuin/.config/atuin/config.toml` | 经过脱敏的 Atuin 通用偏好和过滤规则；需要跨机器复用这些选择时创建，不保存历史、密钥或登录会话（session） | `~/.config/atuin/config.toml` | [[Atuin 命令历史管理]] |
| `scripts/deploy` | 统一封装 Stow 的模拟与应用参数，并限制可部署的 package；第一次手工部署成功后再固化 | 留在仓库中执行，本身不是 Stow package | [[#8. 固化可重复的部署和验证入口]] |
| `scripts/verify` | 检查关键链接是否来自当前仓库、目标目录是否保持为真实目录，并执行最低 Zsh 语法检查 | 留在仓库中执行，不能替代组件运行验证 | [[#8. 固化可重复的部署和验证入口]] |

目标机器专属文件故意不出现在仓库源树中：`local.zprofile`、`local.zsh` 和 `local.ghostty` 是部署后创建的真实文件；`platform.ghostty` 是当前桌面按需选择 `macos.ghostty` 或 `linux.ghostty` 的本机入口。生成的 `.zsh_plugins.zsh`、历史数据库、密钥、session 和缓存同样不属于仓库源。下一节会在这个职责模型上进一步划分 Git 跟踪边界。

不要为了匹配这棵树预建没有实际内容的空配置。平台或组件专题写入第一份有效配置时再创建相应文件；Antidote snapshot 也只在真实生成后进入 Git。

`README.md` 是 fresh clone 后的第一入口，不能长期停留在初始化时的一行标题。首次提交前，至少补齐支持范围、package、前提、部署顺序、验证入口和私有数据边界。例如：

~~~markdown
# Modern terminal dotfiles

面向 macOS、Ubuntu Desktop 和 Ubuntu Server 的终端配置源，由 Git 记录版本，GNU Stow 部署到 `$HOME`。

## Packages

- `zsh`：共享 Shell 配置和 Antidote 声明
- `starship`：提示符配置
- `atuin`：已脱敏的通用偏好；不含历史、密钥和 session
- `ghostty`：仅部署到已安装 Ghostty 的桌面机器

## Deployment

先安装 Git、GNU Stow 和对应组件。对明确的 package 列表运行 `scripts/deploy --simulate`；审查无冲突后，再以相同列表运行 `scripts/deploy --apply`，最后执行 `scripts/verify` 和组件级检查。

## Local-only data

`local.zsh`、`local.zprofile`、`local.ghostty`、历史数据库、密钥、缓存和机器私有路径不进入本仓库，需在目标机器单独重建或从受保护来源恢复。
~~~

README 只能描述仓库真实拥有的 package 和已验证平台；尚未运行过的恢复流程应标为“未验证”，不能写成一键可用。

## 5. 明确进入和不进入仓库的内容

第 4 节解决“仓库中为什么有这些文件”；本节继续解决“哪些内容可以提交，哪些必须留在当前机器或运行时目录”。

| package | 进入 Git 的配置源 | 不进入该 package 的内容 |
| --- | --- | --- |
| `zsh` | `.zshenv`、`.zprofile`、`.zshrc`、插件清单、common 与平台配置、选定 snapshot | `local.zsh`、`local.zprofile`、历史、补全缓存、生成的 `.zsh_plugins.zsh` |
| `ghostty` | 主入口、公共配置、macOS/Linux 模板 | `platform.ghostty`、`local.ghostty`、日志 |
| `starship` | `starship.toml` | Starship 缓存和日志 |
| `atuin` | 审查后的 `config.toml` | 历史数据库、加密密钥、session 和日志 |

本机覆盖文件应在 Stow 部署后的真实目标目录中创建，例如：

~~~bash
umask 077
mkdir -p "$HOME/.config/zsh" "$HOME/.config/ghostty"

[ -e "$HOME/.config/zsh/local.zprofile" ] || : > "$HOME/.config/zsh/local.zprofile"
[ -e "$HOME/.config/zsh/local.zsh" ] || : > "$HOME/.config/zsh/local.zsh"
[ -e "$HOME/.config/ghostty/local.ghostty" ] || : > "$HOME/.config/ghostty/local.ghostty"
~~~

这些命令只在文件不存在时创建空文件，不会覆盖现有内容。它们位于 `$HOME/.config` 的真实目录中，而不是 dotfiles 源树，因此不依赖 `.gitignore` 才能保持不跟踪。

> [!warning] 私有仓库也不是秘密存储
> 不要把令牌、密码、私钥、Atuin 密钥或真实命令历史提交到任何 Git 仓库。`.gitignore` 只影响尚未跟踪的路径；已经进入历史的秘密不能靠新增 ignore 或删除工作区文件消失。若秘密已经提交，应先撤销或轮换，再评估历史清理。

## 6. 从零部署：先模拟，再应用

先按 [[macOS 从零搭建现代终端环境]] 或 [[Ubuntu 从零搭建现代终端环境]] 写入基础 Zsh 配置；需要 Ghostty、Starship 或 Atuin 配置时，再分别按 [[Ghostty 常用配置与 Shell 集成]]、[[Starship 提示符配置]] 和 [[Atuin 命令历史管理]] 创建对应 package 源文件。本文不重复这些配置内容，只从已经存在且经过语法或格式检查的配置源开始部署。准备部署时，预先创建需要容纳符号链接和本机文件的真实目录：

> [!warning] 必须显式选择 package
> 不要在仓库根目录运行 `stow *`。`scripts/` 是仓库工具目录，不是要部署到 `$HOME` 的 Stow package；通配符还可能选中 README 等非 package 内容。每次模拟和应用都应写出同一组明确名称。

~~~bash
mkdir -p \
  "$HOME/.config/zsh" \
  "$HOME/.config/ghostty" \
  "$HOME/.config/atuin"
~~~

macOS 或 Ubuntu Desktop 的基础 package 可以这样模拟：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow \
  --dir="$DOTFILES_DIR" \
  --target="$HOME" \
  --no-folding \
  --simulate \
  --verbose=2 \
  zsh starship atuin ghostty
~~~

无图形界面的 Ubuntu Server 不部署 Ghostty：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow \
  --dir="$DOTFILES_DIR" \
  --target="$HOME" \
  --no-folding \
  --simulate \
  --verbose=2 \
  zsh starship atuin
~~~

模拟只报告计划，不应创建或删除链接。出现 `existing target is neither a link nor a directory` 等 conflict 时应停止，按下一节接管已有配置；不能跳过检查后强制覆盖。

只有模拟结果与预期一致，才去掉 `--simulate` 执行同一组 package：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow \
  --dir="$DOTFILES_DIR" \
  --target="$HOME" \
  --no-folding \
  --verbose=2 \
  zsh starship atuin ghostty
~~~

这段是桌面机器的实际部署命令；Ubuntu Server 必须与前面的服务器模拟保持一致，去掉 `ghostty`。任何机器都不能模拟一组 package、再实际部署另一组未经模拟的 package。

部署成功后，受管理文件应是符号链接，本机覆盖目录仍是真实目录：

~~~bash
for managed_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/atuin/config.toml"
do
  if [ -L "$managed_path" ]; then
    printf 'managed: %s -> %s\n' "$managed_path" "$(readlink "$managed_path")"
  else
    printf 'not a managed link: %s\n' "$managed_path" >&2
  fi
done

for local_directory in "$HOME/.config/zsh" "$HOME/.config/ghostty"; do
  [ -d "$local_directory" ] && [ ! -L "$local_directory" ] \
    && printf 'real directory: %s\n' "$local_directory"
done
~~~

`readlink` 只能证明目标是链接并显示其记录值；还应结合 `DOTFILES_DIR` 和 `git ls-files` 判断它是否由预期仓库管理。

## 7. 安全接管已经存在的配置

已有配置不能直接交给 Stow 覆盖。先列出冲突目标并创建可识别备份：

~~~bash
for existing_path in \
  "$HOME/.zshenv" \
  "$HOME/.config/zsh/.zprofile" \
  "$HOME/.config/zsh/.zshrc" \
  "$HOME/.config/starship.toml" \
  "$HOME/.config/atuin/config.toml"
do
  [ -e "$existing_path" ] || [ -L "$existing_path" ] \
    && ls -ld "$existing_path"
done

backup_dir="$HOME/.terminal-backups/dotfiles-adoption-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"
printf 'backup directory: %s\n' "$backup_dir"
~~~

每个冲突文件都必须先判断“以仓库源为准”还是“以现有目标为准”。以下只处理已经确认是普通文件的 `.zshenv`；若目标是符号链接，应先记录 `readlink` 结果并查明所有者，不能套用普通文件接管流程：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
backup_dir="$HOME/.terminal-backups/dotfiles-adoption-YYYYMMDD-HHMMSS"
source_path="$DOTFILES_DIR/zsh/.zshenv"
target_path="$HOME/.zshenv"

[ -d "$backup_dir" ] || {
  printf '停止：备份目录不存在：%s\n' "$backup_dir" >&2
  exit 1
}
[ -f "$target_path" ] && [ ! -L "$target_path" ] || {
  printf '停止：目标不是待接管的普通文件：%s\n' "$target_path" >&2
  exit 1
}

cp -p "$target_path" "$backup_dir/home.zshenv"
[ ! -e "$source_path" ] || cp -p "$source_path" "$backup_dir/repository.zshenv.before-adoption"
cp -p "$target_path" "$source_path"
cmp -s "$source_path" "$target_path"
~~~

把 `backup_dir` 替换为上一段实际输出。`cmp` 返回成功只能证明两个文件字节相同；仍需阅读内容，确认其中没有代理凭据、令牌或机器专属绝对路径。

若仓库源已经是经过审查的新配置，则不要反向复制旧目标；使用只读差异检查：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

diff -u "$HOME/.zshenv" "$DOTFILES_DIR/zsh/.zshenv" || true
~~~

完成备份、内容选择和秘密审查后，把冲突目标移动到备份目录，再重新执行 Stow 模拟。以下命令只演示一个已经确认的文件，不要未经逐项检查就批量移动整个 `~/.config`：

~~~bash
backup_dir="$HOME/.terminal-backups/dotfiles-adoption-YYYYMMDD-HHMMSS"

[ -d "$backup_dir" ] || {
  printf '停止：备份目录不存在：%s\n' "$backup_dir" >&2
  exit 1
}

if [ -f "$HOME/.zshenv" ] && [ ! -L "$HOME/.zshenv" ]; then
  mv "$HOME/.zshenv" "$backup_dir/home.zshenv.before-stow"
fi
~~~

GNU Stow 提供 `--adopt`，但它会把目标中的现有文件移动进 Stow package，从而直接改变仓库源。本文不把它作为主流程；只有已理解其方向、已保存备份，并准备逐项审查 Git diff 时才可单独评估。

## 8. 固化可重复的部署和验证入口

手工成功部署一次后，可以在仓库中保存一个小型脚本，避免不同笔记复制出不同的 Stow 参数。创建与第一份脚本同时出现的目录：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

mkdir -p "$DOTFILES_DIR/scripts"
~~~

将下列内容保存为 `$DOTFILES_DIR/scripts/deploy`：

~~~sh
#!/bin/sh
set -eu

script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd -P)
repo_dir=$(CDPATH= cd -- "$script_dir/.." && pwd -P)
target_dir=${HOME:?HOME is required}

if [ "$#" -eq 0 ]; then
  printf 'usage: %s [--simulate|--apply] package...\n' "$0" >&2
  exit 2
fi

mode=$1
case "$mode" in
  --simulate)
    action=--simulate
    shift
    ;;
  --apply)
    action=--restow
    shift
    ;;
  *)
    printf 'usage: %s [--simulate|--apply] package...\n' "$0" >&2
    exit 2
    ;;
esac

if [ "$#" -eq 0 ]; then
  printf 'at least one package is required\n' >&2
  exit 2
fi

for package_name in "$@"; do
  case "$package_name" in
    zsh|starship|atuin|ghostty) ;;
    *)
      printf 'unsupported package: %s\n' "$package_name" >&2
      exit 2
      ;;
  esac

  if [ ! -d "$repo_dir/$package_name" ]; then
    printf 'package directory does not exist: %s\n' "$repo_dir/$package_name" >&2
    exit 2
  fi
done

if [ "$action" = --simulate ]; then
  exec stow \
    --dir="$repo_dir" \
    --target="$target_dir" \
    --no-folding \
    --simulate \
    --restow \
    --verbose=2 \
    "$@"
fi

exec stow \
  --dir="$repo_dir" \
  --target="$target_dir" \
  --no-folding \
  --restow \
  --verbose=2 \
  "$@"
~~~

赋予执行权限并先运行模拟：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

chmod u+x "$DOTFILES_DIR/scripts/deploy"
"$DOTFILES_DIR/scripts/deploy" --simulate zsh starship atuin

# 仅在安装 Ghostty 的桌面机器追加 ghostty：
# "$DOTFILES_DIR/scripts/deploy" --simulate zsh starship atuin ghostty
~~~

脚本不会自动安装软件、创建本机 local 文件，也不会猜测当前机器应部署哪些 package；因此 package 列表不能为空。允许列表还会阻止把 `scripts` 误当成 package 部署；以后新增 package 时，应同时审查脚本允许列表和 README。`--simulate` 本身不修改目标；`--apply` 会重新部署明确传入的 package，它应只在相同参数的模拟结果已经审查后执行。

确认模拟输出符合预期后，使用完全相同的 package 列表实际部署：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

"$DOTFILES_DIR/scripts/deploy" --apply zsh starship atuin
~~~

将下列内容保存为 `$DOTFILES_DIR/scripts/verify`，作为链接和 Zsh 语法的最低检查：

~~~sh
#!/bin/sh
set -eu

script_dir=$(CDPATH= cd -- "$(dirname -- "$0")" && pwd -P)
repo_dir=$(CDPATH= cd -- "$script_dir/.." && pwd -P)

check_real_directory() {
  directory_path=$1
  if [ ! -d "$directory_path" ] || [ -L "$directory_path" ]; then
    printf 'expected real directory: %s\n' "$directory_path" >&2
    return 1
  fi
}

check_managed_link() (
  managed_path=$1
  if [ ! -L "$managed_path" ]; then
    printf 'expected symlink: %s\n' "$managed_path" >&2
    return 1
  fi

  if [ ! -e "$managed_path" ]; then
    printf 'broken symlink: %s\n' "$managed_path" >&2
    return 1
  fi

  link_target=$(readlink "$managed_path")
  case "$link_target" in
    /*) link_parent=$(dirname -- "$link_target") ;;
    *) link_parent="$(dirname -- "$managed_path")/$(dirname -- "$link_target")" ;;
  esac
  target_parent=$(CDPATH= cd -- "$link_parent" && pwd -P)
  resolved_path="$target_parent/$(basename -- "$link_target")"

  case "$resolved_path" in
    "$repo_dir"/*) ;;
    *)
      printf 'link is outside repository: %s -> %s\n' "$managed_path" "$resolved_path" >&2
      return 1
      ;;
  esac
)

check_real_directory "$HOME/.config/zsh"
check_managed_link "$HOME/.zshenv"
check_managed_link "$HOME/.config/zsh/.zprofile"
check_managed_link "$HOME/.config/zsh/.zshrc"

if [ -e "$repo_dir/starship/.config/starship.toml" ]; then
  check_managed_link "$HOME/.config/starship.toml"
fi

if [ -e "$repo_dir/atuin/.config/atuin/config.toml" ]; then
  check_real_directory "$HOME/.config/atuin"
  check_managed_link "$HOME/.config/atuin/config.toml"
fi

if [ -e "$repo_dir/ghostty/.config/ghostty/config.ghostty" ]; then
  check_real_directory "$HOME/.config/ghostty"
  check_managed_link "$HOME/.config/ghostty/config.ghostty"
fi

zsh -n "$HOME/.zshenv"
zsh -n "$HOME/.config/zsh/.zprofile"
zsh -n "$HOME/.config/zsh/.zshrc"

printf 'dotfiles managed-link and Zsh syntax checks passed\n'
~~~

然后执行：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

chmod u+x "$DOTFILES_DIR/scripts/verify"
"$DOTFILES_DIR/scripts/verify"
~~~

这个脚本证明三个核心 Zsh 链接来自当前仓库、对应目标目录未被折叠，并在可选 package 的主配置源已经存在时检查 Starship、Atuin 和 Ghostty 链接；它还会解析 Zsh 文件。它不能替代 Antidote、Atuin、Starship、Ghostty 和真实 SSH 会话的运行验证。

## 9. 首次提交与可选远端

运行完整组件验证后，再把当前状态记录为第一个已知良好提交。先检查未跟踪文件和差异：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff --check
git -C "$DOTFILES_DIR" diff
~~~

只暂存已经审查的明确路径，不在包含未知内容时直接使用 `git add -A`：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" add -- README.md zsh scripts

for optional_package in starship atuin ghostty; do
  if [ -d "$DOTFILES_DIR/$optional_package" ] \
    && find "$DOTFILES_DIR/$optional_package" -type f -print -quit | grep -q .; then
    git -C "$DOTFILES_DIR" add -- "$optional_package"
  fi
done

git -C "$DOTFILES_DIR" diff --cached --check
git -C "$DOTFILES_DIR" diff --cached
~~~

重点检查代理地址、令牌、密码、私钥、Atuin 数据、真实主机名和不必要的个人绝对路径。关键词搜索只能作为辅助，不能证明没有秘密：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" grep -nEi \
  'token|password|secret|authorization|private[ _-]?key' \
  -- . || true
~~~

确认暂存内容和组件验证都符合预期后再提交：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

git -C "$DOTFILES_DIR" commit -m "feat: establish modern terminal dotfiles"
git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" log -1 --format=fuller
~~~

远端是跨机器分发和异地备份通道，不是本地 Git 成立的前提。准备连接 GitHub、GitLab 或其他平台时，先按 [[Git 凭据、SSH 与常见问题排查]] 验证认证，然后交互输入远端 URL，避免把个人地址写死在共享教程中：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

printf 'remote repository URL: '
IFS= read -r REPO_URL

git -C "$DOTFILES_DIR" remote add origin "$REPO_URL"
git -C "$DOTFILES_DIR" remote -v
git -C "$DOTFILES_DIR" ls-remote origin HEAD
~~~

确认 URL 指向正确仓库后，再推送当前分支：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
current_branch=$(git -C "$DOTFILES_DIR" branch --show-current)

git -C "$DOTFILES_DIR" push -u origin "$current_branch"
git -C "$DOTFILES_DIR" ls-remote --heads origin "$current_branch"
~~~

仓库可以保持私有，但可见性不能替代秘密隔离。fresh clone 只能得到已经提交并推送的内容；未跟踪、本机覆盖和本地独有提交不会自动出现在另一台机器。

## 10. 日常修改闭环

受管理目标是符号链接，因此编辑 `~/.config/zsh/.zshrc` 会修改仓库源文件。也可以直接在 `DOTFILES_DIR` 中编辑；无论使用哪条入口，开始前都应确认链接归属和 Git 状态。

一次配置调整遵循以下顺序：

1. `git status --short --branch`，识别与本次无关的既有修改。
2. 只修改一个组件或一个清晰主题。
3. 配置内容变化时先运行对应语法与组件检查。
4. 新增、移动或删除源文件时，先执行 `scripts/deploy --simulate`，确认后再 `--apply`。
5. 运行 `scripts/verify` 和组件级验证。
6. 检查 diff，只暂存本次文件，再提交。
7. 配置了远端时再推送；没有远端时明确保留为本地提交。

普通内容修改不会改变符号链接，无需每次都重新 Stow；package 文件集合或路径发生变化时才需要 `--restow`。

## 11. 在新机器恢复

新机器先安装 Git 与 Stow，并确认目标仓库目录不存在。然后输入实际远端 URL，克隆到与当前机器约定一致的 `DOTFILES_DIR`：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

if [ -e "$DOTFILES_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$DOTFILES_DIR" >&2
  exit 1
fi

printf 'remote repository URL: '
IFS= read -r REPO_URL
git clone "$REPO_URL" "$DOTFILES_DIR"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" log -1 --format=oneline
git -C "$DOTFILES_DIR" remote -v
~~~

部署前先检查目标路径是否已经有系统、安装器或人工创建的文件。存在冲突时按第 7 节备份和比较，不直接覆盖。无冲突时：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

"$DOTFILES_DIR/scripts/deploy" --simulate zsh starship atuin

# 安装 Ghostty 的桌面机器另行模拟 ghostty package。
~~~

确认模拟结果符合预期后，再用完全相同的 package 列表实际部署：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

"$DOTFILES_DIR/scripts/deploy" --apply zsh starship atuin

# 安装 Ghostty 的桌面机器另行应用已经模拟过的 ghostty package。
~~~

随后按 [[macOS 从零搭建现代终端环境]] 或 [[Ubuntu 从零搭建现代终端环境]] 安装 Zsh、Antidote、Starship、Atuin、zoxide 和 fzf；共享 Zsh 的文件职责与加载顺序见 [[Zsh 与 Antidote 跨机器配置管理]]。安装完成后创建本机 `local` 文件，让 Antidote 根据清单恢复插件，最后执行：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

"$DOTFILES_DIR/scripts/verify"
zsh -lic 'command -v antidote starship atuin zoxide fzf'
git -C "$DOTFILES_DIR" status --short --branch
~~~

恢复成功至少要求：关键目标仍指向当前 clone、Zsh 语法通过、组件在新会话可见、仓库工作区没有恢复过程产生的意外修改。本机覆盖和 Atuin 私有数据应分别重建或从受保护来源恢复，不能期待 Git 提供。

## 12. 回退与解除部署

### 回退某次配置内容

先保存尚未提交的修改证据，再查看要恢复的提交和文件：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"
backup_dir="$HOME/.terminal-backups/dotfiles-rollback-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$backup_dir"

git -C "$DOTFILES_DIR" status --short --branch
git -C "$DOTFILES_DIR" diff > "$backup_dir/dotfiles-working-tree.patch"
git -C "$DOTFILES_DIR" log --oneline --decorate -10
~~~

已推送的错误提交优先用 `git revert` 生成反向提交，保留历史；尚未提交的单文件实验可在确认备份后，使用带明确路径的 `git restore`。不要用 `git reset --hard` 清理整个仓库。

恢复源文件后再次运行 Stow 模拟、`scripts/verify` 和组件检查。只改内容时链接本身不需要重建。

### 解除 Stow 部署

先模拟删除 package 拥有的链接：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow \
  --dir="$DOTFILES_DIR" \
  --target="$HOME" \
  --no-folding \
  --simulate \
  --delete \
  --verbose=2 \
  zsh starship atuin ghostty
~~~

确认输出只涉及预期链接后，去掉 `--simulate` 执行。Stow 的 `--delete` 表示删除它管理的符号链接，不会删除 package 源文件，也不应删除目标目录中的 `local` 文件：

~~~bash
DOTFILES_DIR="$HOME/.dotfiles"

stow \
  --dir="$DOTFILES_DIR" \
  --target="$HOME" \
  --no-folding \
  --delete \
  --verbose=2 \
  zsh starship atuin ghostty
~~~

只有链接解除后，才把备份中的普通配置复制回原目标。若直接把备份复制到仍然存在的符号链接路径，可能沿链接覆盖 dotfiles 仓库中的源文件。

## 13. 完成标准

本地 dotfiles 最低完成标准：

- `DOTFILES_DIR` 是普通 Git 工作区，并至少有一个经过组件验证的提交；
- `zsh`、`starship`、`atuin` 和按需使用的 `ghostty` package 职责清楚；
- 关键目标是指向该仓库的符号链接，目标配置目录本身不是折叠链接；
- 重复运行相同 package 的 Stow 模拟不会出现意外创建、删除或冲突；
- `git ls-files` 不包含本机覆盖、历史数据库、密钥、session、缓存或生成文件；
- Zsh、Antidote、Starship、Atuin、zoxide、fzf 与 Ghostty 按各自专题通过验证；
- 回退备份仍可读取，并且知道应先解除链接再恢复普通文件。

跨机器完成标准还要求：

- 远端 URL 和当前分支已经核对；
- 当前已验证提交已推送，远端引用可通过 `git ls-remote` 定位；
- 在另一台机器或隔离的测试 HOME 中完成过 clone、模拟部署、实际部署和组件验证；
- 未推送提交、未跟踪文件和本机私有数据被明确记录为“不由 fresh clone 提供”。

## 官方参考资料

- [GNU Stow：官方手册](https://www.gnu.org/software/stow/manual/stow.html)
- [XDG：Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/)
- [Git：gitignore 手册](https://git-scm.com/docs/gitignore)
- [Git：git-add 手册](https://git-scm.com/docs/git-add)
- [GitHub：保护 API 凭据](https://docs.github.com/en/rest/authentication/keeping-your-api-credentials-secure)
- [GitHub：从仓库中移除敏感数据](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [chezmoi：Quick start](https://www.chezmoi.io/quick-start/)
- [chezmoi：跨机器差异管理](https://www.chezmoi.io/user-guide/manage-machine-to-machine-differences/)
- [yadm：Overview](https://yadm.io/docs/overview)
- [yadm：Bootstrap](https://yadm.io/docs/bootstrap)
