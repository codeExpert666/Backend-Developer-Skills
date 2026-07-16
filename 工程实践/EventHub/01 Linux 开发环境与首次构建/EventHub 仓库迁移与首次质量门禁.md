---
title: EventHub 仓库迁移与首次质量门禁
aliases:
  - 在 Linux 中迁移并验证 EventHub Go 与 Java 项目
  - EventHub 项目迁移与首次构建验证
  - Linux EventHub 首次构建
  - EventHub Go 与 Java Linux 验证
tags:
  - 工程实践
  - 工程实践/EventHub
  - EventHub/第1阶段
  - Go/构建
  - Java/Maven
  - Docker/验证
created: 2026-07-16T00:29:35
updated: 2026-07-17T01:13:33
---

本文负责把 EventHub Go 与 Java 仓库放入 Ubuntu 本地文件系统，并按迁移后 revision 的真实工程入口完成首次构建、测试和质量门禁。

通用的 clone、bundle、patch 与 `rsync` 方法见 [[Git 仓库跨机器迁移与工作区保留]]。本文不再复制完整迁移教程，只记录 EventHub 第 1 阶段需要作出的选择、目标目录、项目专属命令和验收边界。

> [!warning] 本文是执行手册，不是成功记录
> 命令、版本和工程入口的核对日期为 **2026-07-16**。仓库之后可能变化；每次执行都必须重新读取 README、Makefile、CI、`go.mod`、`pom.xml` 和 Wrapper。真实 SHA、退出码、PASS/SKIP 与失败原因写入新的日期化执行记录。

## 1. 前置条件

开始前应满足：

- 已按 [[EventHub 第 1 阶段环境与版本基线]] 准备 Ubuntu、Git、Go、JDK、Maven、Docker、Compose 和必要的 Node.js。
- Mac mini 能按 [[OpenSSH 连接、密钥与主机指纹]] 稳定登录虚拟机。
- `$HOME/src` 位于 Ubuntu 本地文件系统，符合 [[Linux 开发工作区与本地文件系统规划]]。
- 已在源机器记录两个仓库的远程、分支、SHA、ahead/behind 和工作区状态。
- 已区分远端可获得内容与只存在于源机器的工作现场。

本阶段的目标目录固定为：

```text
$HOME/src/
├── eventhub-go/
└── eventhub/
```

这里的 `$HOME` 属于 Ubuntu 开发用户。不要沿用 macOS 绝对路径，也不要把项目长期放在 UTM 共享挂载中。

## 2. 为两个仓库分别选择迁移路线

不要因为两个项目属于同一产品，就强行使用相同迁移方式。

| 源仓库状态 | 本阶段路线 |
| --- | --- |
| 所需提交已推送，且不需要本地额外内容 | 在 Ubuntu fresh clone |
| 有未推送提交，但允许推送 | 整理、验证、提交并推送，再 clone |
| 有未推送提交，不能推送 | 用 Git bundle 搬运提交历史 |
| 有可审查的未提交修改 | 先提交，或用二进制安全 patch |
| 必须保留 `.git`、未跟踪文件和完整现场 | 受控 `rsync` 到全新暂存目录，再验证切换 |

准备检查发现的动态 Git 状态见 [[2026-07-16 第 1 阶段准备检查记录]]。真正迁移前必须重查，历史记录不能替代当前事实。

### 2.1 源机器检查模板

对 Go 和 Java 仓库分别执行。

**执行位置：macOS 宿主机（任意目录，只读）**

```bash
printf '请输入要检查的源仓库绝对路径：'
IFS= read -r SOURCE_REPO

if ! git -C "$SOURCE_REPO" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
  printf '停止：不是可访问的 Git 工作树：%s\n' "$SOURCE_REPO" >&2
  exit 1
fi

git -C "$SOURCE_REPO" status --short --branch
git -C "$SOURCE_REPO" branch --show-current
git -C "$SOURCE_REPO" remote -v
git -C "$SOURCE_REPO" rev-parse HEAD
git -C "$SOURCE_REPO" branch -vv
git -C "$SOURCE_REPO" diff --check
git -C "$SOURCE_REPO" ls-files --others --exclude-standard
```

如果输出暴露未公开文件名，只保存在个人执行记录中，不复制到通用或公开笔记。

### 2.2 核对关键工程入口是否被跟踪

**执行位置：macOS 宿主机（Go 项目根目录）**

```bash
for FILE_NAME in README.md Makefile go.mod Dockerfile docker-compose.yml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done

if [ -d .github/workflows ]; then
  git ls-files '.github/workflows/*'
fi
```

**执行位置：macOS 宿主机（Java 项目根目录）**

```bash
for FILE_NAME in README.md Makefile pom.xml mvnw mvnw.cmd \
  .mvn/wrapper/maven-wrapper.properties Dockerfile docker-compose.yml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done

if [ -d .github/workflows ]; then
  git ls-files '.github/workflows/*'
fi
```

出现 `local-only` 时，fresh clone 不会得到该文件。不能在 Ubuntu 中手工创建同名空文件来“补齐入口”，应回到 [[Git 仓库跨机器迁移与工作区保留]] 选择可审计的迁移方法。

## 3. 迁移后建立 Linux 端身份基线

先确认目录位于本地文件系统。

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
findmnt --target "$HOME/src"
df -hT "$HOME/src"
namei -l "$HOME/src"
```

`findmnt` 应显示 Ubuntu 自己的本地文件系统，而不是共享挂载。目录链上的用户和权限应允许当前开发用户读写。

再分别核对两个仓库：

**执行位置：Ubuntu 虚拟机（任意目录）**

```bash
for PROJECT_NAME in eventhub-go eventhub; do
  PROJECT_DIR="$HOME/src/$PROJECT_NAME"
  printf '\n[%s]\n' "$PROJECT_NAME"

  if ! git -C "$PROJECT_DIR" rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    printf '失败：不是 Git 工作树：%s\n' "$PROJECT_DIR" >&2
    exit 1
  fi

  git -C "$PROJECT_DIR" remote -v
  git -C "$PROJECT_DIR" branch --show-current
  git -C "$PROJECT_DIR" rev-parse HEAD
  git -C "$PROJECT_DIR" status --short --branch
  git -C "$PROJECT_DIR" diff --check
done
```

如果采用 fresh clone，HEAD 应与源机器记录的已推送提交一致，工作区通常应干净。如果采用完整工作现场迁移，目标状态应与源机器事实快照一致，而不是强行清空。

## 4. 重新读取当前 revision 的工程入口

迁移成功后，不凭记忆直接运行旧命令。

### 4.1 Go 项目

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
pwd
git status --short --branch
git rev-parse HEAD
sed -n '1,100p' go.mod
sed -n '1,240p' README.md
sed -n '1,260p' Makefile

if [ -d .github/workflows ]; then
  find .github/workflows -maxdepth 1 -type f -print
  sed -n '1,280p' .github/workflows/*.yml
fi

git ls-files README.md Makefile go.mod Dockerfile docker-compose.yml \
  '.github/workflows/*'
```

重点确认：

- `go.mod` 的 `go` 和可选 `toolchain` 指令。
- README 的前置条件。
- Make target 的实际 recipe 和是否会写入文件。
- CI 中的 Go、Node 和容器依赖。
- OpenAPI、sqlc、lint、测试与 Docker 入口。

### 4.2 Java 项目

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub`）**

```bash
pwd
git status --short --branch
git rev-parse HEAD
sed -n '1,240p' README.md
sed -n '1,520p' pom.xml

for FILE_NAME in Makefile mvnw mvnw.cmd \
  .mvn/wrapper/maven-wrapper.properties .github/workflows/ci.yml; do
  if git ls-files --error-unmatch "$FILE_NAME" >/dev/null 2>&1; then
    printf 'tracked    %s\n' "$FILE_NAME"
  elif [ -e "$FILE_NAME" ]; then
    printf 'local-only %s\n' "$FILE_NAME"
  else
    printf 'absent     %s\n' "$FILE_NAME"
  fi
done
```

重点确认：

- POM 声明的 JDK 版本和 Maven profiles。
- 当前 revision 是否真正包含 Wrapper。
- README、POM、Makefile 和 CI 中哪个才是可复现入口。
- Checkstyle、SpotBugs 等插件是否已在当前 POM 中配置。

## 5. EventHub Go 首次构建与门禁

Go 安装见 [[Ubuntu 安装 Go]]，精确版本见 [[EventHub 第 1 阶段环境与版本基线]]。

### 5.1 核对工具链与模块位置

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
go version
go env GOMOD GOPATH GOCACHE GOOS GOARCH GOTOOLCHAIN
awk '$1 == "go" { print "go directive=" $2; exit }' go.mod
awk '$1 == "toolchain" { print "toolchain directive=" $2; exit }' go.mod
```

`GOMOD` 应指向当前项目的 `go.mod`，`GOOS` 应为 `linux`，架构应与客户机一致。若工具链不满足项目声明，停止并修正环境，不要修改 `go.mod` 来迁就本机。

### 5.2 依赖下载与原生构建

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
git status --short --branch
go mod download
go build ./...
git status --short --branch
```

`go build ./...` 证明包能够编译，但不会自动证明测试、静态分析、生成物和容器构建都通过。

### 5.3 在干净 checkout 上执行质量总入口

> [!warning] 当前 `quality-check` 不是纯只读
> 核对日期对应的 Makefile 中，格式检查会先运行写入式 `gofmt`，再检查 Git diff。只有在干净且可恢复的 checkout 中执行；不要在混有未提交开发内容的工作区直接运行。

如果本阶段需要验证源机器上的未提交修改，应先把待验证状态整理成可恢复的本地检查点，例如独立分支上的提交并同时保存 bundle；再在该提交的干净 checkout 中运行门禁。直接把修改 stash 后测试只会验证旧 HEAD，不能证明这些修改通过。

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
if [ -n "$(git status --porcelain)" ]; then
  printf '%s\n' '停止：quality-check 前工作区必须干净。' >&2
  exit 1
fi

make quality-check
git status --short --branch
```

当前入口应覆盖格式、`go vet`、测试和 golangci-lint。若以后 Makefile 变化，以实际 recipe 和 CI 为准。成功后工作区应仍然干净。

核对日期对应的 Makefile 会优先使用版本匹配的本机 golangci-lint；工具缺失或版本不匹配时，回退到固定 Docker 镜像。因此即使格式、vet 和普通测试不依赖容器，完整 `quality-check` 仍可能要求 Docker daemon 可用。

### 5.4 证明 MySQL 集成测试没有被跳过

当前 repository 集成测试使用 Testcontainers。Docker provider 不健康时，测试可能显示 `SKIP` 而整体命令仍成功，因此必须单独观察输出。

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
docker info >/dev/null
go test -v ./internal/repository/mysql \
  -run '^TestMySQLPersistenceFoundation$' \
  -count=1
```

验收要求是指定测试显示 `PASS`，不是只看到总退出码为 0，也不能接受因为 Docker 不可用而 `SKIP`。

### 5.5 OpenAPI 与生成物门禁

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
node --version
npx --version
make openapi-lint
```

Node 主版本以当前 CI 为准。不要通过安装任意“最新版”来掩盖版本不一致。

`generated-check` 会运行生成器并用 Git diff 检查漂移，也必须在干净 checkout 中执行。

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
if [ -n "$(git status --porcelain)" ]; then
  printf '%s\n' '停止：generated-check 前工作区必须干净。' >&2
  exit 1
fi

make generated-check
git status --short --branch
```

如果生成物出现差异，应保留 diff 判断是输入、生成器版本还是已提交生成物漂移；不要为了让门禁变绿而未经审查地提交或删除差异。

PR 场景还要对真实目标分支运行 breaking check。

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
printf '请输入真实的 PR 目标分支：'
IFS= read -r BASE_BRANCH

git fetch --no-tags origin \
  "refs/heads/${BASE_BRANCH}:refs/remotes/origin/${BASE_BRANCH}"
git rev-parse --verify "origin/$BASE_BRANCH^{commit}"
make openapi-breaking-check OPENAPI_BASE_REF="origin/$BASE_BRANCH"
```

如果当前不是 PR 验证场景，在执行记录中写明“不适用”，不要虚构已完成。

### 5.6 Compose 与镜像构建

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub-go`）**

```bash
docker compose version
docker compose config --quiet
docker info >/dev/null
make docker-build
docker image inspect eventhub-go:local \
  --format '{{.Id}} {{.Architecture}}'
```

镜像标签必须以当前 Makefile 为准。Compose 静态解析成功不等于容器已启动或应用健康；`make docker-build` 也不等于已经部署。

## 6. EventHub Java 首次构建与门禁

Java 与 Maven 安装、版本切换和仓库配置分别见：

- [[Java 与 Maven 环境搭建概览]]
- [[Ubuntu 安装 Java 与 Maven]]
- [[Java 版本管理与环境变量配置]]
- [[Maven 常用配置与仓库管理]]
- [[Java 与 Maven 环境排障与维护]]

### 6.1 核对 JDK 与 Maven 实际使用版本

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub`）**

```bash
java -version
javac -version
mvn -version 2>/dev/null || true
grep -nE 'java\.version|maven\.compiler\.release' pom.xml
git status --short --branch
```

`java`、`javac` 和 Maven 输出中的 Java home 必须满足当前 POM。若 Maven 使用了不同 JDK，应先处理 `PATH`、`JAVA_HOME` 或 `update-alternatives`，再开始构建。

### 6.2 fresh clone 路线

若当前 revision 没有跟踪 Wrapper 或额外质量入口，只能使用 README 和 POM 确实声明的全局 Maven 命令。

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub`）**

```bash
mvn test -Ptest
mvn package -Pprod
docker compose config --quiet
find target -maxdepth 1 -type f -name 'backend-*.jar' -print
git status --short --branch
```

预期 Maven 输出 `BUILD SUCCESS`，测试通过，`target` 中出现预期 Jar，Compose 配置解析成功。

> [!important] H2 + Flyway 不等于真实 MySQL
> 测试 profile 使用 H2 兼容模式时，可以验证 Spring 上下文、基础 SQL 和迁移链路，但不能覆盖所有 MySQL 类型、索引、锁、排序规则和事务行为。真实 MySQL/Redis 验证必须按当前项目入口单独执行和记录。

### 6.3 完整工作区迁移路线

只有当 Wrapper、Makefile、CI 和质量配置确实随本次迁移到达，并且来源可追溯时，才使用它们。

**执行位置：Ubuntu 虚拟机（`$HOME/src/eventhub`）**

```bash
for FILE_NAME in Makefile mvnw \
  .mvn/wrapper/maven-wrapper.properties .github/workflows/ci.yml; do
  if [ ! -e "$FILE_NAME" ]; then
    printf '停止：缺少质量入口文件：%s\n' "$FILE_NAME" >&2
    exit 1
  fi
done

if [ ! -x ./mvnw ]; then
  printf '%s\n' '停止：mvnw 不可执行。' >&2
  exit 1
fi

./mvnw --version
make -n ci
make ci
git status --short --branch
```

`make -n ci` 先展示实际 recipe。若文件缺失，回到迁移选择，不创建同名空文件；若门禁失败，按测试、Checkstyle、SpotBugs、Compose 或 Wrapper 下载阶段分别排查，不使用 `-DskipTests` 统一掩盖问题。

## 7. 结束前做前后状态对照

**执行位置：Ubuntu 虚拟机（各项目根目录）**

```bash
git status --short --branch
git branch --show-current
git remote -v
git rev-parse HEAD
git diff --check
```

执行记录至少保存：

1. Ubuntu、CPU 架构和工具版本。
2. 两个仓库的分支、完整 SHA、上游和工作区状态。
3. 实际执行命令及退出码。
4. Go MySQL 集成测试是 `PASS` 还是 `SKIP`。
5. Java 使用 fresh clone Maven 路线还是完整工作区质量入口。
6. Compose 只做静态验证，还是还做了镜像构建或运行时冒烟。
7. 门禁产生的工作区变化及处理方式。
8. 未执行、不适用或失败的项目及原因。

## 8. 常见失败与安全恢复

| 现象 | 优先判断 | 安全处理 |
| --- | --- | --- |
| clone 后 HEAD 不一致 | 提交未推送、分支错误或远端已变化 | 停止构建，重新核对 SHA；不使用 `reset --hard` 掩盖差异 |
| clone 后缺少工程入口 | 文件未跟踪或当前 revision 本来不存在 | 用 `git ls-files` 证明，再选择正确迁移路线 |
| 项目目录不可写 | 使用过 `sudo` 或复制后所有权异常 | 按 [[Linux 用户、用户组、sudo 与文件权限]] 核对真实目标后最小修复 |
| Go 总测试成功但集成测试 `SKIP` | Docker provider 不健康 | 先让 `docker info` 成功，再单独运行指定测试 |
| `quality-check` 或生成器修改文件 | 门禁本身包含写入式命令 | 保留并审查 diff；只有执行前确认为干净基线时才恢复 |
| Java 无法运行 `./mvnw` | 当前 revision 没有 Wrapper或执行位丢失 | 使用真实 README 入口，或重新验证完整迁移 |
| Compose 配置成功但 build/up 失败 | 静态解析不覆盖 daemon、架构、端口和健康检查 | 分层检查 Docker、镜像和服务日志 |

## 9. 本篇完成标准

详细退出条件由 [[EventHub 第 1 阶段验收清单]] 统一维护。本篇至少应完成：

- [ ] 两个仓库位于 Ubuntu 本地 `$HOME/src`。
- [ ] 每个仓库的迁移路线与原因已记录。
- [ ] 远程、分支、完整 SHA 和工作区状态已核对。
- [ ] Go 项目完成当前 revision 的构建、测试和质量门禁。
- [ ] Go MySQL 集成测试明确显示 `PASS`。
- [ ] Java 项目按当前 revision 的真实入口完成测试与打包。
- [ ] 两个项目的 Compose 配置完成静态验证。
- [ ] 验证前后 Git 状态已对照并解释。
- [ ] 没有把构建、镜像构建或 Compose 静态解析描述成部署成功。

验证通过后，继续 [[UTM 虚拟机快照、备份与恢复]]，再回到 [[EventHub 第 1 阶段验收清单]] 收口。

## 官方资料

- [Go command 官方参考](https://pkg.go.dev/cmd/go)
- [Go Modules Reference](https://go.dev/ref/mod)
- [Docker Compose config](https://docs.docker.com/reference/cli/docker/compose/config/)
- [Docker buildx build](https://docs.docker.com/reference/cli/docker/buildx/build/)
- [Apache Maven Build Lifecycle](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Apache Maven Wrapper](https://maven.apache.org/tools/wrapper/)
- [Node.js 官方下载](https://nodejs.org/en/download)
- [Redocly CLI 官方文档](https://redocly.com/docs/cli)
