---
title: 在 Linux 中迁移并验证 EventHub Go 与 Java 项目
aliases:
  - EventHub 项目迁移与首次构建验证
  - Linux EventHub 首次构建
  - EventHub Go 与 Java Linux 验证
tags:
  - Linux
  - Linux/开发环境
  - Git/迁移
  - Go/构建
  - Java/Maven
  - Docker/验证
created: 2026-07-16T00:29:35
updated: 2026-07-16T01:17:59
---

本文负责把已经准备好的 EventHub Go 与 Java 仓库安全迁移到 Ubuntu 虚拟机的本地文件系统，并完成第一次可追溯的构建、测试和质量门禁验证。它不是一份“复制两个目录，然后看到命令不报错就结束”的清单：迁移前必须先判断远端是否已经包含全部工作，迁移后必须核对分支、远程地址、提交 SHA、工具版本与当前 revision 实际提供的工程入口。

开始前应已完成 [[Linux 后端开发虚拟机搭建概览]]、[[使用 UTM 创建 Ubuntu Server 开发虚拟机]]、[[Ubuntu Server 开发机初始化与安全基线]]、[[从 macOS 使用 SSH 连接 Linux 虚拟机]] 和 [[Linux 后端开发目录与工具链规划]]。外出时可以经 [[使用 Tailscale 远程访问 Linux 开发机]] 连接，但 Tailscale 不是本次迁移和首次构建的必要条件。验证成功后，再按 [[Linux 开发虚拟机备份恢复与常见问题]] 建立恢复基线。

> [!important] 本文中的“预期通过”不等于已经在虚拟机中执行成功
> 2026-07-16 的案例数据来自对 macOS 上两个仓库的只读检查，用于选择迁移路线和确定命令。真正的 Linux 验收结果必须由你在 Ubuntu 虚拟机中执行命令后记录，不能用本文示例代替。

## 1. 本篇在第 1 阶段中的位置

完整阅读顺序如下：

| 顺序 | 笔记 | 本篇需要用到的结果 |
| --- | --- | --- |
| 1 | [[Linux 后端开发虚拟机搭建概览]] | 理解阶段目标、边界和完成清单 |
| 2 | [[使用 UTM 创建 Ubuntu Server 开发虚拟机]] | Ubuntu Server ARM64 虚拟机能够启动和联网 |
| 3 | [[Ubuntu Server 开发机初始化与安全基线]] | 用户、`sudo`、时间、软件包与基础权限正常 |
| 4 | [[从 macOS 使用 SSH 连接 Linux 虚拟机]] | Mac mini 可以通过 SSH 密钥稳定登录虚拟机 |
| 5 | [[使用 Tailscale 远程访问 Linux 开发机]] | 可选的 MacBook Air 外出访问路径 |
| 6 | [[Linux 后端开发目录与工具链规划]] | Git、Go、JDK、Docker 与 Compose 已按项目约束准备 |
| 7 | [[在 Linux 中迁移并验证 EventHub Go 与 Java 项目]] | 当前这一步：迁移、核对、构建和测试 |
| 8 | [[Linux 开发虚拟机备份恢复与常见问题]] | 为已经验证的开发机建立快照与恢复基线 |

本篇只验证 Linux 开发环境。它不包含云服务器部署、正式 CI/CD、生产监控、生产数据库迁移、发布审批或 Kubernetes。

## 2. 先理解四个容易混淆的边界

### 2.1 最终工作目录必须位于 Linux 本地文件系统

在 Ubuntu 虚拟机中统一使用：

```text
$HOME/src/
├── eventhub-go/
└── eventhub/
```

这里的 `$HOME` 是 Ubuntu 用户自己的主目录，不是 macOS 的 `/Users/...`，也不是 UTM 共享目录。

| 位置 | 适合做什么 | 不适合做什么 |
| --- | --- | --- |
| Ubuntu 本地 `$HOME/src` | 长期开发、Git 操作、编译、测试、容器构建 | 无 |
| UTM 共享挂载 | 临时交换安装包或导出文件 | 长期保存活跃仓库、数据库目录、容器卷 |
| macOS iCloud 目录 | 文档同步和归档 | Linux 虚拟机内的活跃源码工作树 |

Linux 本地文件系统能稳定表达 Linux 权限、符号链接、大小写、文件监听和可执行位。共享挂载还会把宿主机生命周期和文件系统语义带进开发环境，因此只能作为传输通道，不能成为项目长期运行位置。

### 2.2 `git clone` 只会得到远端已经知道的内容

`git clone` 会复制远端可达的提交和引用，并创建新的工作树。它不会自动带来下面这些内容：

- 本地尚未 push 的提交。
- 已修改但尚未 commit 的文件内容。
- 未跟踪文件。
- 被 `.gitignore` 忽略的本地配置、缓存或密钥。
- 只存在于原机器 `.git/config` 中的本地设置。

因此，“本地仓库能构建”与“fresh clone 能复现”是两个不同结论。

### 2.3 构建、测试和质量门禁不是同一个动作

| 名称 | 要回答的问题 | EventHub 示例 |
| --- | --- | --- |
| 依赖下载 | 依赖能否按声明解析到本地缓存 | `go mod download`、Maven 首次解析依赖 |
| 构建 | 源码能否编译并产生可运行或可打包的产物 | `go build ./...`、`mvn package` |
| 测试 | 自动化用例是否通过 | `go test ./...`、`mvn test -Ptest` |
| 静态分析 | 不运行程序也能发现的规则或缺陷是否通过 | `go vet`、golangci-lint、Checkstyle、SpotBugs |
| 生成物检查 | 规范、SQL 与生成代码是否同步 | `make generated-check` |
| 契约检查 | OpenAPI 是否满足语法、风格和兼容性规则 | `make openapi-lint` 等 |
| 容器构建 | Dockerfile 和基础镜像能否产生镜像 | `make docker-build` |
| Compose 静态验证 | Compose 配置能否解析和归一化 | `docker compose config --quiet` |
| 运行时冒烟 | 容器、依赖和应用能否真正启动并响应 | 项目 README 或 Makefile 中的 smoke/up 入口 |

> [!warning] Compose 静态验证通过不等于服务启动成功
> `docker compose config --quiet` 主要验证配置解析，不会证明 Docker daemon 正常、镜像支持 ARM64、端口未冲突、数据库能启动或应用健康检查能通过。

### 2.4 原生运行与容器运行验证的是不同边界

- 原生 Go/JDK 构建验证 Ubuntu 中的工具链、源码和依赖缓存。
- Docker 镜像构建验证 Dockerfile、构建上下文、基础镜像和 daemon。
- Compose 验证进一步覆盖多个服务之间的配置和网络关系。

第 1 阶段应先完成原生构建，再做容器配置和镜像验证；不要因为容器能编译，就跳过 Linux 本地 Go/JDK 的版本核对。

## 3. 迁移前先建立源仓库事实快照

先完成 [[Git 安装与初始配置概览]]、[[Ubuntu 从零安装 Git]] 和 [[Git 常用配置与本地验证]]。访问代码托管平台时遇到 SSH 或 HTTPS 认证问题，阅读 [[Git 凭据、SSH 与常见问题排查]]。

> [!important] 两类 SSH 不要混为一谈
> `ssh eventhub-dev-vm` 是登录 Ubuntu 开发机；`git clone` 使用的 SSH URL 则是 Git 客户端向代码托管平台认证。它们都使用 SSH 协议和密钥概念，但服务端、主机指纹、密钥授权位置和用途不同。

### 3.1 对每个源仓库执行只读检查

下面以 Go 仓库为例。检查 Java 仓库时只需替换 `SOURCE_DIR`。

**执行位置：macOS 宿主机（任意目录）**

```bash
SOURCE_DIR="$HOME/Programming/Code/Go/src/eventhub-go"

git -C "$SOURCE_DIR" status --short --branch
git -C "$SOURCE_DIR" branch --show-current
git -C "$SOURCE_DIR" remote -v
git -C "$SOURCE_DIR" rev-parse HEAD
git -C "$SOURCE_DIR" rev-parse --abbrev-ref --symbolic-full-name '@{upstream}'
git -C "$SOURCE_DIR" rev-parse '@{upstream}'
git -C "$SOURCE_DIR" rev-list --left-right --count '@{upstream}...HEAD'
git -C "$SOURCE_DIR" diff --check
git -C "$SOURCE_DIR" ls-files --others --exclude-standard
```

这组命令分别回答：工作树是否干净、当前分支是什么、远程地址是什么、本地 HEAD 是哪个提交、上游引用是什么、本地相对上游领先或落后多少，以及是否存在未跟踪文件。

预期不是“输出必须为空”，而是你能解释每一行：

- `status --short --branch` 只有分支行，表示工作树干净。
- `rev-list --left-right --count` 输出两个数字；对 `上游...HEAD` 而言，左侧是上游独有提交数，右侧是本地独有提交数。
- `diff --check` 无输出且退出码为 0，表示当前差异没有 Git 能识别的空白错误。
- `ls-files --others --exclude-standard` 无输出，表示没有未跟踪且未忽略的文件。

如果 `@{upstream}` 不存在，先用 `git branch -vv` 确认该分支是否本来就没有上游，不要随意把它绑定到猜测的远程分支。如果输出含敏感文件名，只在本机处理，不要把完整状态粘贴进公开笔记或聊天。

### 3.2 判断关键工程文件是否真的被 Git 跟踪

文件“存在”不等于 fresh clone 后“仍存在”。

**执行位置：macOS 宿主机（Go 项目根目录）**

```bash
for FILE_NAME in README.md Makefile go.mod .github/workflows/ci.yml Dockerfile docker-compose.yml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done
```

**执行位置：macOS 宿主机（Java 项目根目录）**

```bash
for FILE_NAME in README.md Makefile pom.xml mvnw mvnw.cmd .mvn/wrapper/maven-wrapper.properties .github/workflows/ci.yml Dockerfile docker-compose.yml config/checkstyle/checkstyle.xml config/spotbugs/exclude-filter.xml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done
```

看到 `local-only` 时，不能继续假设 clone 后能使用该文件提供的命令。恢复方式不是在 Linux 中手工造一个同名空文件，而是回到本节选择提交、bundle、patch 或受控复制。

### 3.3 本次实操案例快照

以下状态核对日期为 **2026-07-16**，只描述迁移决策需要的事实，不公开具体未提交业务内容。

| 项目 | 项目约束与提交状态 | 对迁移方式的影响 |
| --- | --- | --- |
| EventHub Go | `go.mod` 声明 Go `1.24.0`；本地 `main` HEAD 为 `45e80f9b814ce9937be9b3120ed48efdac8e1fda`；远端 `main` 为 `40213fc51e454e3662935948138dc555a618125c`；本地领先 1 个提交且仍有未提交修改 | fresh clone 只能到远端提交，不能复现完整本地工作区；至少需要先 push、本地提交迁移加 patch，或受控 `rsync` |
| EventHub Java | POM 声明 JDK 21；本地与远端 `main` 都是 `b11a06bca0e74d33266fd2b2c83c685779df9ec0`；工作区有未提交内容和仅本地存在的工程文件 | fresh clone 不包含当前本地 Makefile、Maven Wrapper、CI 与静态分析配置；clone 路线只能使用远端 revision 实际声明的 Maven 入口 |

这些 SHA 是本次案例的审计证据，不是以后永久不变的安装参数。真正操作时始终重新运行上一节的命令。

## 4. 情况一：本地工作已经提交并推送

这是最容易验证、最适合作为长期主线的路线。

### 4.1 在源仓库记录 clone 参数

**执行位置：macOS 宿主机（项目根目录）**

```bash
REPO_URL="$(git remote get-url origin)"
BRANCH_NAME="$(git branch --show-current)"
EXPECTED_SHA="$(git rev-parse HEAD)"

printf '远程地址：%s\n' "$REPO_URL"
printf '分支：%s\n' "$BRANCH_NAME"
printf '完整 SHA：%s\n' "$EXPECTED_SHA"
git status --short --branch
```

预期分支非空、SHA 为 40 位十六进制字符串，并且工作树干净。若工作树不干净，说明“已提交并推送”的前提不成立，应转到第 5 节。

### 4.2 clone 到 Linux 本地目录

先为项目准备父目录。目标项目目录必须不存在，借此避免把 clone 结果混进旧目录。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
mkdir -p "$HOME/src"
PROJECT_NAME=eventhub-go
TARGET_DIR="$HOME/src/$PROJECT_NAME"

if [ -e "$TARGET_DIR" ]; then
  printf '停止：目标已经存在：%s\n' "$TARGET_DIR" >&2
  exit 1
fi

printf '请输入源仓库核对过的远程地址：'
IFS= read -r REPO_URL
printf '请输入要检出的分支名：'
IFS= read -r BRANCH_NAME

git clone --branch "$BRANCH_NAME" --single-branch "$REPO_URL" "$TARGET_DIR"
```

检查 Java 项目时设置 `PROJECT_NAME=eventhub`。`--single-branch` 让首次迁移聚焦当前分支；以后仍可通过明确的 fetch refspec 获取其他分支。

成功时会创建目标目录并检出指定分支。认证失败时，不要把密码或令牌写进命令历史；根据远程 URL 类型回到 [[Git 凭据、SSH 与常见问题排查]]。clone 中断后先保留现场检查；如果需要重试，把不完整目录改名留档，再创建新的空目标，禁止直接向半成品目录覆盖复制。

### 4.3 核对远程、分支和完整 SHA

**执行位置：Ubuntu 虚拟机（项目父目录）**

```bash
PROJECT_NAME=eventhub-go
TARGET_DIR="$HOME/src/$PROJECT_NAME"

printf '请输入源仓库记录的完整 SHA：'
IFS= read -r EXPECTED_SHA

git -C "$TARGET_DIR" remote -v
git -C "$TARGET_DIR" branch --show-current
git -C "$TARGET_DIR" status --short --branch
ACTUAL_SHA="$(git -C "$TARGET_DIR" rev-parse HEAD)"
REMOTE_SHA="$(git -C "$TARGET_DIR" rev-parse '@{upstream}')"

printf '本地 SHA：%s\n' "$ACTUAL_SHA"
printf '上游 SHA：%s\n' "$REMOTE_SHA"

if [ "$ACTUAL_SHA" != "$EXPECTED_SHA" ]; then
  printf '失败：Linux HEAD 与源仓库记录不一致。\n' >&2
  exit 1
fi

if [ "$ACTUAL_SHA" != "$REMOTE_SHA" ]; then
  printf '失败：Linux HEAD 与当前上游不一致。\n' >&2
  exit 1
fi
```

两个比较都通过，才能证明“源端记录的提交、Linux 当前 HEAD、Linux 当前上游”一致。失败时优先检查是否输入了错误分支、源端提交是否真的 push、远程默认分支是否变化；不要用 `reset --hard` 强行让输出看起来一致。

## 5. 情况二：存在未推送提交或未提交修改

按下面顺序选择，越靠前越容易审计：

1. 整理工作，提交并 push，再按第 4 节 clone。
2. 用 Git bundle 迁移尚未 push 的提交和引用。
3. 用 `format-patch` / `git am` 迁移一组线性提交。
4. 用二进制 patch 迁移已跟踪文件的未提交差异。
5. 用受控 `rsync` 复制完整工作树，包括 `.git`、隐藏文件和未跟踪文件。

### 5.1 首选：整理后提交并 push

先交互式选择要提交的块，避免 `git add -A` 把缓存、临时文件或其他任务一起带入。

**执行位置：macOS 宿主机（项目根目录）**

```bash
git status --short --branch
git diff --check
git add --patch
git diff --cached --stat
git diff --cached --check
git commit

BRANCH_NAME="$(git branch --show-current)"
git push --set-upstream origin "$BRANCH_NAME"
git status --short --branch

REMAINING_STATUS="$(git status --porcelain)"
if [ -n "$REMAINING_STATUS" ]; then
  printf '%s\n' '停止转入 clone 路线：push 后仍有未提交或未跟踪内容。' >&2
  printf '%s\n' "$REMAINING_STATUS"
  false
else
  printf '%s\n' '工作区已清空，可以按第 4 节重新 clone 并核对 SHA。'
fi
```

提交前必须阅读 staged diff，而不是只看文件数量。若选错内容，可以用 `git restore --staged :/` 只取消暂存，不会删除工作树修改；重新检查后再暂存。push 失败时保留本地提交，先修复认证或网络，不要重复创建相同提交。最后的 guard 退出码非 0 时，说明仍有内容不会随 clone 出现，应继续提交、patch 或受控复制，不能假装迁移已完成。

### 5.2 Git bundle：迁移提交和引用

Bundle 适合传递未 push 的本地提交、分支和标签；它不包含工作树未提交修改，也不包含未跟踪文件。

#### 在源仓库创建并校验 bundle

**执行位置：macOS 宿主机（任意目录）**

```bash
SOURCE_DIR="$HOME/Programming/Code/Go/src/eventhub-go"
BUNDLE_FILE="$HOME/Desktop/eventhub-go-stage1.bundle"
BUNDLE_HEAD_FILE="$BUNDLE_FILE.head"

if [ -e "$BUNDLE_FILE" ] || [ -e "$BUNDLE_HEAD_FILE" ]; then
  printf '停止：bundle 或 HEAD 清单已经存在：%s\n' "$BUNDLE_FILE" >&2
  exit 1
fi

EXPECTED_BUNDLE_HEAD="$(git -C "$SOURCE_DIR" rev-parse HEAD)"
git -C "$SOURCE_DIR" bundle create "$BUNDLE_FILE" --all
git -C "$SOURCE_DIR" bundle verify "$BUNDLE_FILE"
printf '%s\n' "$EXPECTED_BUNDLE_HEAD" > "$BUNDLE_HEAD_FILE"
shasum -a 256 "$BUNDLE_FILE"
printf 'bundle_expected_head=%s\n' "$EXPECTED_BUNDLE_HEAD"
```

预期 `bundle verify` 成功，并输出可记录的 SHA-256。失败时不要传输文件；先运行 `git fsck` 和 `git show-ref` 检查源仓库引用。

#### 通过已经验证的 SSH 连接传入虚拟机

**执行位置：macOS 宿主机（任意目录）**

```bash
SSH_TARGET=eventhub-dev-vm
BUNDLE_FILE="$HOME/Desktop/eventhub-go-stage1.bundle"
BUNDLE_HEAD_FILE="$BUNDLE_FILE.head"

if ssh "$SSH_TARGET" 'set -eu
  mkdir -p "$HOME/incoming"
  test ! -e "$HOME/incoming/eventhub-go-stage1.bundle"
  test ! -e "$HOME/incoming/eventhub-go-stage1.bundle.head"'; then
  scp "$BUNDLE_FILE" "$BUNDLE_HEAD_FILE" "${SSH_TARGET}:incoming/"
else
  printf '%s\n' '停止：远端同名 bundle 或 HEAD 清单已经存在。' >&2
fi
```

`eventhub-dev-vm` 是 [[从 macOS 使用 SSH 连接 Linux 虚拟机]] 中建立的 `~/.ssh/config` 别名，不是固定主机名。传输后在 Linux 使用 `sha256sum`，确认结果与 macOS 的 `shasum -a 256` 完全相同。

#### 在 fresh clone 中引入 bundle 的提交

下面假定已经按第 4 节 clone 了远端基线，这样 `origin` 仍指向真正代码托管平台。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
BUNDLE_FILE="$HOME/incoming/eventhub-go-stage1.bundle"
BUNDLE_HEAD_FILE="$BUNDLE_FILE.head"
BRANCH_NAME=main

sha256sum "$BUNDLE_FILE"
git bundle verify "$BUNDLE_FILE"
EXPECTED_SHA="$(tr -d '\r\n' < "$BUNDLE_HEAD_FILE")"
if printf '%s\n' "$EXPECTED_SHA" | grep -Eq '^[0-9a-f]{40}$'; then
  git fetch "$BUNDLE_FILE" "refs/heads/$BRANCH_NAME:refs/remotes/bundle/$BRANCH_NAME"
  git log --oneline --decorate --graph --max-count=12 --all
  git switch "$BRANCH_NAME"
  git merge --ff-only "bundle/$BRANCH_NAME"
  git status --short --branch
  ACTUAL_SHA="$(git rev-parse HEAD)"
  printf 'expected=%s\nactual=%s\n' "$EXPECTED_SHA" "$ACTUAL_SHA"
  test "$ACTUAL_SHA" = "$EXPECTED_SHA"
else
  printf '%s\n' '停止：bundle HEAD 清单不是 40 位 SHA。' >&2
  false
fi
```

如果本地分支只是远端分支的后继，`--ff-only` 会安全快进；若历史已经分叉，它会拒绝修改。此时不要强制 merge，可以先用下面的命令把 bundle 侧历史隔离成新分支：

**执行位置：Ubuntu 虚拟机（项目根目录）**

```bash
BRANCH_NAME=main
git switch -c "migrated-$BRANCH_NAME" "bundle/$BRANCH_NAME"
git status --short --branch
```

确认提交完整后再决定如何整合。Bundle 文件本身可以在阶段备份完成后删除，但删除前要确认提交已经存在于正常 Git 引用或远端中。

### 5.3 `format-patch` 与 `git am`：迁移线性提交

这种方式把每个提交导出为邮件格式 patch，并在目标仓库重建作者、提交消息和 diff。它适合少量线性提交；复杂分支、merge commit 或多引用状态优先使用 bundle。

#### 导出尚未 push 的提交

**执行位置：macOS 宿主机（项目根目录）**

```bash
PATCH_DIR="$HOME/Desktop/eventhub-go-format-patches"

if [ -e "$PATCH_DIR" ]; then
  printf '停止：patch 目录已经存在：%s\n' "$PATCH_DIR" >&2
  exit 1
fi

mkdir -p "$PATCH_DIR"
BASE_SHA="$(git rev-parse '@{upstream}')"
EXPECTED_SHA="$(git rev-parse HEAD)"
printf 'BASE_SHA=%s\nEXPECTED_SHA=%s\n' "$BASE_SHA" "$EXPECTED_SHA" \
  > "$PATCH_DIR/migration-manifest.txt"
git log --oneline '@{upstream}'..HEAD
git format-patch --output-directory "$PATCH_DIR" '@{upstream}'..HEAD
find "$PATCH_DIR" -maxdepth 1 -type f -name '*.patch' -print
cat "$PATCH_DIR/migration-manifest.txt"
```

预期每个待迁移提交对应一个 patch，manifest 固定导出时的上游基线与源端最终 HEAD。没有 patch 输出时先确认本地是否真的领先上游。`format-patch` 不会包含未提交修改。

#### 传输并应用

**执行位置：macOS 宿主机（任意目录）**

```bash
SSH_TARGET=eventhub-dev-vm
PATCH_DIR="$HOME/Desktop/eventhub-go-format-patches"

if ssh "$SSH_TARGET" 'set -eu
  mkdir -p "$HOME/incoming"
  test ! -e "$HOME/incoming/eventhub-go-format-patches"'; then
  scp -r "$PATCH_DIR" "${SSH_TARGET}:incoming/"
else
  printf '%s\n' '停止：远端同名 format-patch 目录已经存在。' >&2
fi
```

**执行位置：Ubuntu 虚拟机（Go 项目根目录，必须是干净的远端基线）**

```bash
PATCH_DIR="$HOME/incoming/eventhub-go-format-patches"
MANIFEST_FILE="$PATCH_DIR/migration-manifest.txt"

if [ -n "$(git status --porcelain)" ]; then
  printf '停止：目标工作树不干净。\n' >&2
  exit 1
fi

BASE_SHA="$(awk -F= '$1 == "BASE_SHA" { print $2 }' "$MANIFEST_FILE")"
EXPECTED_SHA="$(awk -F= '$1 == "EXPECTED_SHA" { print $2 }' "$MANIFEST_FILE")"
ACTUAL_BASE="$(git rev-parse HEAD)"

if [ "$ACTUAL_BASE" != "$BASE_SHA" ]; then
  printf '停止：目标基线与导出 patch 时的上游 SHA 不一致。\n' >&2
  printf 'expected_base=%s\nactual_base=%s\n' "$BASE_SHA" "$ACTUAL_BASE"
  false
else
  git am "$PATCH_DIR"/*.patch
  git status --short --branch
  git log --oneline --decorate --max-count=12
  ACTUAL_SHA="$(git rev-parse HEAD)"
  printf 'expected=%s\nactual=%s\n' "$EXPECTED_SHA" "$ACTUAL_SHA"
  test "$ACTUAL_SHA" = "$EXPECTED_SHA"
fi
```

冲突时 Git 会暂停在 `am` 状态。先阅读冲突和 patch；决定放弃本次应用时执行 `git am --abort`，它会回到 `git am` 开始前，而不是删除原有分支历史。

### 5.4 二进制 patch：迁移已跟踪文件的未提交差异

先迁移提交，再对那个准确 HEAD 生成工作树 patch。`--binary` 能保留 Git 已跟踪二进制内容和可执行位变化，但仍不会包含未跟踪文件。

#### 生成 patch 并记录基线 SHA

**执行位置：macOS 宿主机（Go 项目根目录）**

```bash
PATCH_FILE="$HOME/Desktop/eventhub-go-worktree.patch"
BASE_SHA="$(git rev-parse HEAD)"

if [ -e "$PATCH_FILE" ]; then
  printf '停止：patch 文件已经存在：%s\n' "$PATCH_FILE" >&2
  exit 1
fi

git diff --check "$BASE_SHA" -- .
git diff --binary "$BASE_SHA" -- . > "$PATCH_FILE"
test -s "$PATCH_FILE"
printf '基线 SHA：%s\n' "$BASE_SHA"
shasum -a 256 "$PATCH_FILE"
git ls-files --others --exclude-standard
```

最后一条命令专门列出 patch 没有覆盖的未跟踪文件。必须逐项决定提交、单独传输或改用 `rsync`，不能误以为 patch 已经包含全部工作区。

#### 传输工作树 patch

**执行位置：macOS 宿主机（任意目录）**

```bash
SSH_TARGET=eventhub-dev-vm
PATCH_FILE="$HOME/Desktop/eventhub-go-worktree.patch"

if ssh "$SSH_TARGET" 'set -eu
  mkdir -p "$HOME/incoming"
  test ! -e "$HOME/incoming/eventhub-go-worktree.patch"'; then
  scp "$PATCH_FILE" "${SSH_TARGET}:incoming/"
else
  printf '%s\n' '停止：远端同名工作树 patch 已经存在。' >&2
fi
```

传输后在 macOS 用 `shasum -a 256`、在 Ubuntu 用 `sha256sum` 核对同一个 patch 文件；两端摘要必须完全一致，再继续应用。

#### 在相同基线应用

**执行位置：Ubuntu 虚拟机（项目根目录）**

```bash
PATCH_FILE="$HOME/incoming/eventhub-go-worktree.patch"

printf '请输入源仓库记录的基线 SHA：'
IFS= read -r BASE_SHA

if [ -n "$(git status --porcelain)" ]; then
  printf '停止：目标工作树必须是干净的准确基线。\n' >&2
  exit 1
elif [ "$(git rev-parse HEAD)" != "$BASE_SHA" ]; then
  printf '停止：目标 HEAD 不是 patch 的基线。\n' >&2
  exit 1
fi

git apply --check "$PATCH_FILE"
git apply "$PATCH_FILE"
git status --short --branch
git diff --check
```

目标必须同时满足“HEAD 等于源端记录基线”和“工作树干净”；否则即使 patch 能应用，也无法证明结果准确复现源工作区。`git apply --check` 先做无写入校验。应用后的差异默认处于未暂存状态，源端原有的 staged / unstaged 划分不会由 patch 保留。

应用后若发现传错 patch，且之后没有继续编辑，可先运行 `git apply --check --reverse "$PATCH_FILE"`，确认能反向应用后再执行 `git apply --reverse "$PATCH_FILE"`。如果已经继续修改，应保留现场并人工处理，不能盲目反向覆盖。

### 5.5 受控 `rsync`：复制完整工作树

当本地同时有未 push 提交、未提交修改、未跟踪文件以及隐藏工程文件时，`rsync` 可以完整复制工作树和 `.git`。它也是风险最高的路线，因为忽略文件可能包含缓存、密钥或机器专属配置。

#### 先审计复制范围

**执行位置：macOS 宿主机（Go 或 Java 项目根目录）**

```bash
git status --short --branch
git status --short --ignored
git ls-files --others --exclude-standard
git ls-files --others --ignored --exclude-standard
du -sh . .git
```

逐项确认忽略文件中没有不应离开宿主机的令牌、私钥、数据库转储或超大缓存。不要把这组输出发布到公开位置。

#### 强制使用全新的空目标目录

**执行位置：macOS 宿主机（任意目录）**

```bash
SSH_TARGET=eventhub-dev-vm
TARGET_NAME=eventhub-go

ssh "$SSH_TARGET" 'set -eu
TARGET_DIR="$HOME/src/'"$TARGET_NAME"'"
if [ -e "$TARGET_DIR" ]; then
  printf "停止：目标已经存在：%s\n" "$TARGET_DIR" >&2
  exit 1
fi
mkdir -p "$TARGET_DIR"'
```

如果目标已经存在，命令会停止。正确做法是核对旧目录身份后为本次迁移选择新的时间戳目录或先把旧目录改名归档，不要启用 `--delete`，也不要覆盖已有工作区。

#### 冻结源工作区并先 dry-run

`rsync` 复制的是一段时间内持续变化的目录，不是原子快照。开始前先关闭会写入项目的 IDE 保存任务、生成器、测试、构建和文件监听器；从记录基线到目标验证完成期间不要继续编辑。若无法暂停写入，应改用先生成归档或先提交的路线。

**执行位置：macOS 宿主机（任意目录）**

```bash
SOURCE_DIR="$HOME/Programming/Code/Go/src/eventhub-go"
SSH_TARGET=eventhub-dev-vm
TARGET_NAME=eventhub-go

SOURCE_HEAD_BEFORE=$(git -C "$SOURCE_DIR" rev-parse HEAD)
SOURCE_STATUS_BEFORE=$(git -C "$SOURCE_DIR" status --porcelain=v1 --untracked-files=all)

if ! rsync -a --no-owner --no-group --dry-run --itemize-changes \
  "$SOURCE_DIR/" \
  "${SSH_TARGET}:src/${TARGET_NAME}/"; then
  printf '停止：dry-run 失败，没有开始实际复制。\n' >&2
  exit 1
fi
```

末尾的 `/` 表示复制源目录的内容，而不是在目标内再套一层 `eventhub-go`。`-a` 会递归复制并保留常规权限位、时间、符号链接和隐藏文件，包括 `.git`；`--no-owner --no-group` 避免把 macOS 的用户和组编号强行带到 Linux，目标所有者应是登录 Ubuntu 的开发用户。

逐行检查 `--itemize-changes` 输出，确认目标只有预期新增项，没有错误目录层级和敏感文件。上面的 Shell 变量只在当前会话有效，因此实际复制应继续在同一个终端中进行。dry-run 通过后，先确认源仓库基线未变、目标仍为空，再执行实际复制；完成后第二次 dry-run 必须没有输出：

**执行位置：macOS 宿主机（同一个终端会话）**

```bash
printf '确认源工作区已暂停写入，并执行复制？输入 COPY：'
IFS= read -r CONFIRM

if [ "$CONFIRM" != COPY ]; then
  printf '已取消，没有复制文件。\n'
  exit 0
fi

if [ "$(git -C "$SOURCE_DIR" rev-parse HEAD)" != "$SOURCE_HEAD_BEFORE" ] || \
   [ "$(git -C "$SOURCE_DIR" status --porcelain=v1 --untracked-files=all)" != "$SOURCE_STATUS_BEFORE" ]; then
  printf '停止：源仓库 HEAD 或工作区状态已经变化，请重新审计。\n' >&2
  exit 1
fi

if ! ssh "$SSH_TARGET" sh -s -- "$TARGET_NAME" <<'REMOTE'
set -eu
TARGET_DIR="$HOME/src/$1"
if [ ! -d "$TARGET_DIR" ]; then
  printf '停止：目标目录不存在。\n' >&2
  exit 1
fi
if [ -n "$(find "$TARGET_DIR" -mindepth 1 -maxdepth 1 -print -quit)" ]; then
  printf '停止：目标目录已经包含文件。\n' >&2
  exit 1
fi
REMOTE
then
  printf '停止：无法证明远端目标存在且为空，不执行实际复制。\n' >&2
  exit 1
fi

if ! rsync -a --no-owner --no-group --itemize-changes \
  "$SOURCE_DIR/" \
  "${SSH_TARGET}:src/${TARGET_NAME}/"; then
  printf '停止：实际复制失败；目标是未验证半成品，不要继续使用。\n' >&2
  exit 1
fi

POSTCHECK=$(mktemp)
trap 'rm -f "$POSTCHECK"' EXIT
if ! rsync -a --no-owner --no-group --checksum --delete \
  --dry-run --itemize-changes \
  "$SOURCE_DIR/" \
  "${SSH_TARGET}:src/${TARGET_NAME}/" >"$POSTCHECK"; then
  printf '停止：复制后复核命令失败；目标仍未验证。\n' >&2
  exit 1
fi

if [ -s "$POSTCHECK" ]; then
  printf '警告：复制后仍有差异；不要开始开发，先检查以下清单：\n' >&2
  cat "$POSTCHECK" >&2
  exit 1
fi

printf '复制后复核无差异；继续在 Linux 端验证 Git 状态。\n'
```

复制后的第二次 dry-run 使用 `--checksum` 比较文件内容，并用仅限 dry-run 的 `--delete` 把目标端多余项显示为待删除项；由于同时存在 `--dry-run`，它不会实际删除文件。这能发现普通文件内容变化、源端删除后目标残留等 rsync 范围内的差异，但仍不是原子快照，也不证明所有扩展属性或运行中数据库状态一致。

若最后仍有差异，应把目标视为未验证副本，重新冻结源目录后再迁移。若源工作区无法停止写入，应先创建不可变归档，或组合使用 Git bundle、patch 与单独审计的未跟踪文件，而不是把一次运行中的目录复制描述成完整快照。

#### 在 Linux 验证复制结果

**执行位置：Ubuntu 虚拟机（项目父目录）**

```bash
TARGET_DIR="$HOME/src/eventhub-go"

git -C "$TARGET_DIR" fsck --full
git -C "$TARGET_DIR" status --short --branch
git -C "$TARGET_DIR" remote -v
git -C "$TARGET_DIR" branch --show-current
git -C "$TARGET_DIR" rev-parse HEAD
git -C "$TARGET_DIR" diff --check
```

预期 Git 对象库完整，状态、分支、HEAD 和源端一致。源端与目标端都运行 `git status --porcelain=v1`，逐行比较修改、未跟踪文件和可执行位。

如果传输中断，不要向可能混有旧文件的目标反复覆盖。可以先把半成品改名保留证据，再重新创建空目标：

**执行位置：macOS 宿主机（任意目录）**

```bash
SSH_TARGET=eventhub-dev-vm
TARGET_NAME=eventhub-go

ssh "$SSH_TARGET" 'set -eu
TARGET_DIR="$HOME/src/'"$TARGET_NAME"'"
ARCHIVE_DIR="$TARGET_DIR.incomplete.$(date +%Y%m%d%H%M%S)"
mv "$TARGET_DIR" "$ARCHIVE_DIR"
printf "半成品已保留为：%s\n" "$ARCHIVE_DIR"'
```

确认重新迁移成功且不再需要半成品后，再人工决定是否删除归档目录。

## 6. 迁移后先读取当前 revision 的工程入口

不要凭记忆运行命令。先检查迁移后的真实文件，并再次确认这些文件是否被当前提交跟踪。

### 6.1 Go 项目

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
git status --short --branch
git rev-parse HEAD
git remote -v
sed -n '1,80p' go.mod
sed -n '1,220p' README.md
sed -n '1,220p' Makefile

if [ -d .github/workflows ]; then
  find .github/workflows -maxdepth 1 -type f -print
fi

git ls-files README.md Makefile go.mod .github/workflows Dockerfile docker-compose.yml
```

重点确认：`go` 指令、README 前置条件、Make target、CI 使用的 Go/Node 版本、Docker 镜像目标和生成代码检查。文件缺失时以该 revision 为准，不要从另一台机器的记忆拼凑命令。

### 6.2 Java 项目

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
git status --short --branch
git rev-parse HEAD
git remote -v
sed -n '1,220p' README.md
sed -n '1,460p' pom.xml

for FILE_NAME in Makefile mvnw mvnw.cmd .mvn/wrapper/maven-wrapper.properties .github/workflows/ci.yml config/checkstyle/checkstyle.xml config/spotbugs/exclude-filter.xml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done
```

重点确认：`java.version`、Maven profile、Wrapper 是否存在且可执行、Makefile 实际 recipe、CI 质量入口、Checkstyle/SpotBugs 是否真的在当前 POM 中绑定。

## 7. EventHub Go 首次构建与质量验证

Go 安装和升级方法见 [[Ubuntu 安装 Go]]。本次 revision 的 `go.mod` 声明 `go 1.24.0`；项目约束优先于系统仓库默认版本。

### 7.1 核对 Go 工具链和模块位置

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
go version
go env GOMOD GOPATH GOCACHE GOOS GOARCH
```

预期 `GOMOD` 指向 `$HOME/src/eventhub-go/go.mod`，`GOOS` 为 `linux`，Apple Silicon 对应的 Ubuntu ARM64 通常显示 `GOARCH=arm64`。`go version` 必须满足当前 `go.mod`；如果不满足，停止构建并回到 [[Ubuntu 安装 Go]]，不要靠修改 `go.mod` 规避版本要求。

### 7.2 下载依赖并完成原生构建

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
go mod download
go build ./...
git status --short --branch
```

成功时两条 Go 命令退出码为 0。`go build ./...` 会编译所有包并使用 Go build cache，不等于已经执行测试。依赖下载失败时先检查 DNS、系统时间、代理环境和 `go env GOPROXY`；不要随意改成来源不明的代理。构建后 Git 状态出现变化时先检查 diff，不能继续假定仓库仍是干净基线。

### 7.3 在干净 checkout 上执行 `make quality-check`

> [!warning] `fmt-check` 会先写文件
> 当前 Makefile 的 `fmt-check` 先执行 `gofmt -w .`，然后用 `git diff --exit-code` 检查漂移。它不是纯只读检查。只应在已经确认干净、可恢复的 checkout 中运行；带未提交开发内容的工作树应先提交、创建安全副本或改用隔离 worktree。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
if [ -n "$(git status --porcelain)" ]; then
  printf '停止：quality-check 前工作树必须干净。\n' >&2
  exit 1
fi

make quality-check
git status --short --branch
```

当前入口按顺序执行 `fmt-check -> vet -> test -> lint`。本机缺少项目要求版本的 golangci-lint 时，Makefile 会回退到固定 Docker 镜像，因此即使前几步不需要容器，lint 仍可能要求 Docker daemon 可用。

预期所有 target 退出码为 0，结束后 Git 工作树仍干净。若 `fmt-check` 留下 diff，先阅读 `git diff -- '*.go'`；这些差异说明源码与 `gofmt` 结果不一致。只有在运行前已经确认 checkout 干净、且明确要放弃生成的格式变化时，才可以对这些具体 Go 文件使用 `git restore`，不要对包含真实开发内容的工作树批量恢复。

### 7.4 确认 MySQL 集成测试没有被跳过

当前 repository 集成测试使用 Testcontainers，并调用 `SkipIfProviderIsNotHealthy`。因此 `go test ./...` 整体成功时，Docker 不健康可能导致该测试显示 `SKIP`，而不是失败。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
docker info >/dev/null
go test -v ./internal/repository/mysql \
  -run '^TestMySQLPersistenceFoundation$' \
  -count=1
```

必须在输出中看到 `TestMySQLPersistenceFoundation` 的 `PASS`，不能只看最终退出码，也不能接受 `SKIP`。该测试会启动 MySQL 容器、执行 migration up/down 并验证 repository 行为。

如果 `docker info` 失败，先按 [[Ubuntu 安装 Docker]] 检查 daemon、systemd 状态和当前用户权限；重新登录用户会话后再试。如果测试拉取镜像失败，检查 DNS、代理、磁盘空间和镜像架构，不要把跳过测试当作通过。

### 7.5 OpenAPI lint 与生成物检查

当前 CI 为 OpenAPI lint 提供 Node.js 24，Makefile 通过固定版本的 Redocly CLI 执行 `npx`。先按 [[Linux 后端开发目录与工具链规划]] 核对 Node，而不是盲目安装“最新版”。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
node --version
npx --version
make openapi-lint
```

预期 Node 主版本符合当前 CI，lint 无 error。第一次运行 `npx` 可能下载固定版本工具并写入用户缓存；失败时先检查 npm registry 连通性和磁盘权限，不要把临时工具提交进仓库。

`generated-check` 会实际运行 sqlc 与 OpenAPI 生成器，再用 Git diff 检查生成物是否漂移，因此也必须在干净 checkout 上运行。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
if [ -n "$(git status --porcelain)" ]; then
  printf '停止：generated-check 前工作树必须干净。\n' >&2
  exit 1
fi

make generated-check
git status --short --branch
```

预期生成命令成功且最终无 diff。失败时保留并检查 `git diff -- internal/repository/mysql/sqlc api/openapi/gen`，判断是工具版本、输入规范还是已提交生成物漂移；不要为了让门禁变绿而不加审查地提交生成结果。

#### PR 基线的 OpenAPI breaking check

当前 CI 还在 pull request 场景比较 PR 目标分支与当前 `api/openapi/eventhub.yaml`。Makefile 固定使用 `oasdiff v1.21.0`，只在 `/api/v1` 及其子路径中出现 `ERR` 级 breaking change 时失败；它不依赖 Node.js，但首次 `go run` 可能下载对应 Go module。

目标本身不会刷新 remote-tracking ref。执行前先 fetch，并把 `BASE_BRANCH` 设置为真实 PR 目标分支；普通 `main` 基线示例如下：

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
BASE_BRANCH=main

git fetch --no-tags origin \
  "refs/heads/${BASE_BRANCH}:refs/remotes/origin/${BASE_BRANCH}"
git rev-parse --verify "origin/$BASE_BRANCH^{commit}"
make openapi-breaking-check OPENAPI_BASE_REF="origin/$BASE_BRANCH"
```

前文使用 `--single-branch` clone 时，普通 `git fetch origin` 可能只更新当初选择的分支；这里显式把真实 base branch 写入对应的 `origin/<base>` remote-tracking ref。在 GitHub Actions 中，CI 同样使用 PR 的实际 base branch，而不是永久写死 `main`。

该检查把 base ref 中的契约作为旧版本、当前工作区文件作为新版本，因此未提交的 OpenAPI 修改也会参与比较。若 fetch 失败、base ref 不存在、base commit 中没有契约文件，目标会停止；不要用过期的 `origin/main` 让检查看似通过。

### 7.6 Compose 静态验证和镜像构建

Docker 概念与安装边界见 [[Docker 安装概览]] 和 [[Ubuntu 安装 Docker]]。

**执行位置：Ubuntu 虚拟机（Go 项目根目录）**

```bash
docker compose version
docker compose config --quiet
docker info >/dev/null
make docker-build
docker image inspect eventhub-go:local --format '{{.Id}} {{.Architecture}}'
```

预期 Compose 配置解析成功，Docker daemon 可访问，`make docker-build` 生成 Makefile 当前默认标签的镜像。镜像检查应显示镜像 ID 和与当前构建平台相符的架构。

若 Makefile 已修改默认标签，应以实际 Makefile 为准。Compose 配置通过但镜像构建失败时，分别检查 daemon、构建上下文、基础镜像 ARM64 支持、网络和磁盘；不要把静态验证结果写成“项目容器已启动”。

## 8. EventHub Java 首次构建与验证

先阅读 [[Java 与 Maven 环境搭建概览]] 和 [[Ubuntu 安装 Java 与 Maven]]。多 JDK 切换见 [[Java 版本管理与环境变量配置]]，仓库、Wrapper 和代理见 [[Maven 常用配置与仓库管理]]，故障处理见 [[Java 与 Maven 环境排障与维护]]。

### 8.1 fresh clone 当前远端 revision 的入口

2026-07-16 检查时，远端 revision 声明 Java 21 和 Maven 3.9+，但没有跟踪当前 macOS 工作区中仅本地存在的 Makefile、Maven Wrapper、CI 与静态分析配置。因此 fresh clone 必须先使用该 revision README 的 Maven 入口，不能直接写 `make ci` 或 `./mvnw`。

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
java -version
javac -version
mvn -version
git status --short --branch
```

预期 `java`、`javac` 和 Maven 实际使用的 Java 都是 21。若 Maven 显示另一个 Java home，按 [[Java 版本管理与环境变量配置]] 检查 `PATH`、`JAVA_HOME` 和 `update-alternatives`，不要只修改一处后立即假定所有终端都生效。

### 8.2 测试、打包和 Compose 静态验证

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
mvn test -Ptest
mvn package -Pprod
docker compose config --quiet
find target -maxdepth 1 -type f -name 'backend-*.jar' -print
git status --short --branch
```

预期 Maven 输出 `BUILD SUCCESS`，测试通过，`target` 下存在项目 Jar，Compose 配置退出码为 0。`target/` 通常被 Git 忽略，因此构建后 Git 工作树应保持与构建前相同。

> [!important] H2 + Flyway 测试不等于真实 MySQL 验证
> 当前 `test` profile 使用 H2 内存数据库并运行 Flyway，能验证测试上下文和迁移链路的基础兼容性，但 H2 的 MySQL 模式不能覆盖所有 MySQL 类型、索引、锁、排序规则和事务行为。真实 MySQL/Redis 冒烟应按当前 README/Makefile 的容器入口单独执行和记录。

依赖解析失败时先检查 Maven 使用的 JDK、`~/.m2/settings.xml`、网络和磁盘；测试失败时保留 Surefire 报告，不要直接改成跳过测试。Compose 静态验证失败时查看错误指出的字段和环境变量，不要直接启动服务试图绕过解析错误。

### 8.3 只有完整迁移本地质量文件后才能使用 Wrapper/Makefile

如果选择了受控 `rsync`，或先把这些工程文件正式提交并迁移，必须先确认所有入口真实存在，再运行质量门禁。

**执行位置：Ubuntu 虚拟机（Java 项目根目录）**

```bash
for FILE_NAME in Makefile mvnw .mvn/wrapper/maven-wrapper.properties .github/workflows/ci.yml config/checkstyle/checkstyle.xml config/spotbugs/exclude-filter.xml; do
  if [ ! -e "$FILE_NAME" ]; then
    printf '停止：缺少质量入口文件：%s\n' "$FILE_NAME" >&2
    exit 1
  fi
done

if [ ! -x ./mvnw ]; then
  printf '停止：mvnw 不可执行。\n' >&2
  exit 1
fi

grep -q 'maven-checkstyle-plugin' pom.xml
grep -q 'spotbugs-maven-plugin' pom.xml
./mvnw --version
make -n ci
make ci
git status --short --branch
```

2026-07-16 的本地案例 Wrapper 固定 Maven 3.9.16；以后应以迁移后的 `.mvn/wrapper/maven-wrapper.properties` 为准。`make -n ci` 先展示将执行的命令，便于确认它确实对应当前 revision；当前本地入口会先静态校验 Compose，再通过 Wrapper 执行 `clean verify`、测试、Checkstyle 和 SpotBugs。

如果任何文件缺失，停止并回到迁移选择，不要创建同名空文件。若 `make ci` 失败，按输出区分 Compose 解析、Wrapper 下载、测试、Checkstyle 和 SpotBugs；这些失败代表不同问题，不能用 `-DskipTests` 或禁用规则统一掩盖。

## 9. 结束前做前后状态对照

对两个仓库分别执行，并把输出记录到本阶段验收记录中。

**执行位置：Ubuntu 虚拟机（各项目根目录）**

```bash
git status --short --branch
git branch --show-current
git remote -v
git rev-parse HEAD
git diff --check
```

如果目标是验证 fresh clone，结束时应仍然干净；如果目标是完整迁移未提交工作区，结束状态应与源端事实快照一致，而不是强行为空。

建议记录：

- Ubuntu 版本和 `uname -m`。
- Go、Java、Maven、Docker、Compose、Node 版本。
- 两个仓库的分支、完整 HEAD SHA 和上游 SHA。
- 每个验证命令、执行时间、退出结果。
- Go MySQL 集成测试是 `PASS` 还是 `SKIP`。
- Java 使用的是 fresh clone Maven 入口还是完整迁移后的 Wrapper/Makefile。
- Compose 只做了静态验证，还是进一步完成了镜像构建或运行时冒烟。
- 任何被跳过的门禁及原因。

## 10. 常见失败与安全处理

| 现象 | 优先判断 | 安全处理与恢复 |
| --- | --- | --- |
| clone 后 HEAD 与源端不一致 | 源端提交未 push、分支输错、远端变化 | 停止构建，重新核对完整 SHA 和上游；不要 `reset --hard` 掩盖差异 |
| clone 后缺少 Makefile、Wrapper 或 CI | 文件只存在源端工作区或当前 revision 没有跟踪 | 用 `git ls-files` 证明状态，再选择提交、bundle、patch 或 `rsync` |
| `rsync` 目标出现旧文件 | 目标不是空目录或路径层级错误 | dry-run 阶段停止；改名归档旧目录，重新创建空目标；不使用 `--delete` |
| Git 报 dubious ownership 或无法写入 | 项目所有者不是当前 Ubuntu 用户 | 用 `id`、`ls -ld` 核对来源；修正目标归属，不要把整个系统目录改成宽松权限 |
| Go 集成测试整体成功但显示 `SKIP` | Docker provider 不健康 | 先通过 `docker info`，再单独以 `-v` 运行指定测试并确认 `PASS` |
| `make quality-check` 修改 Go 文件 | `fmt-check` 实际执行了 `gofmt -w` | 阅读 diff；只有运行前确认干净时才恢复具体文件，否则保留并人工合并 |
| `make generated-check` 留下 diff | 输入规范、生成器版本或提交生成物发生漂移 | 保留 diff 定位来源；不要直接提交或批量 restore |
| Java fresh clone 不能运行 `./mvnw` | 当前远端 revision 没有 Wrapper | 使用该 revision README 声明的全局 Maven 入口，或重新迁移完整工程文件 |
| Maven 测试通过但 MySQL 问题仍存在 | H2 测试边界被误当成真实 MySQL | 单独执行项目声明的真实依赖冒烟，不修改测试结果描述 |
| Compose 配置通过但 build/up 失败 | 静态解析没有覆盖 daemon、架构、端口和健康检查 | 分层检查 `docker info`、镜像架构、构建日志、容器状态与服务日志 |

## 11. 本篇完成标准

- [ ] 两个仓库都位于 Ubuntu 本地 `$HOME/src`，不在共享挂载或 iCloud 中。
- [ ] 已明确每个仓库采用 clone、bundle、patch 还是受控 `rsync`。
- [ ] 已记录源端和 Linux 端的分支、远程地址与完整 SHA。
- [ ] 未把 fresh clone 无法获得的本地文件误写成远端基线。
- [ ] EventHub Go 的 `go version` 满足当前 `go.mod`。
- [ ] EventHub Go 完成 `go mod download` 和 `go build ./...`。
- [ ] EventHub Go 在干净 checkout 上完成 `make quality-check`。
- [ ] EventHub Go 完成 `make openapi-lint` 与 `make generated-check`，或明确记录无法执行的原因。
- [ ] EventHub Go 已针对真实 PR base ref 完成 `make openapi-breaking-check`，或明确记录当前不是 PR 验证场景。
- [ ] EventHub Go 的 MySQL 集成测试明确显示 `PASS`，没有被 Docker 不健康静默跳过。
- [ ] EventHub Go 完成 `docker compose config --quiet` 和 `make docker-build`。
- [ ] EventHub Java 使用 JDK 21，并按当前 revision 的实际 README 入口完成测试与打包。
- [ ] 若 Java 使用本地 Wrapper/Makefile，已先证明相关文件完整存在且来源可追溯。
- [ ] 已明确 Java 的 H2 + Flyway 测试不等于真实 MySQL 验证。
- [ ] 两个仓库在验证前后的 `git status` 已对照并解释。
- [ ] 已区分原生构建、容器镜像构建、Compose 静态验证和运行时冒烟。
- [ ] 已准备进入 [[Linux 开发虚拟机备份恢复与常见问题]]，没有把第 1 阶段扩张成部署阶段。

## 12. 官方参考资料

以下链接核对日期为 **2026-07-16**。项目版本和命令仍以迁移后 revision 的 README、Makefile、CI、`go.mod` 与 `pom.xml` 为准。

- [Git clone 官方手册](https://git-scm.com/docs/git-clone)
- [Git bundle 官方手册](https://git-scm.com/docs/git-bundle)
- [Git format-patch 官方手册](https://git-scm.com/docs/git-format-patch)
- [Git am 官方手册](https://git-scm.com/docs/git-am)
- [Git diff 官方手册](https://git-scm.com/docs/git-diff)
- [rsync 官方文档入口](https://rsync.samba.org/documentation.html)
- [Go Modules Reference](https://go.dev/ref/mod)
- [Go command 官方参考](https://pkg.go.dev/cmd/go)
- [Docker Compose config 官方参考](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker buildx build 官方参考](https://docs.docker.com/reference/cli/docker/buildx/build/)
- [Apache Maven Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Apache Maven Wrapper](https://maven.apache.org/tools/wrapper/)
- [Node.js 官方下载与版本入口](https://nodejs.org/en/download)
- [Redocly CLI 官方文档](https://redocly.com/docs/cli)
