---
title: macOS 安装 Java 与 Maven
aliases:
  - Mac 安装 Java 与 Maven
  - macOS 安装 JDK 与 Maven
tags:
  - Java
  - Java/安装
  - Java/macOS
  - Maven
created: 2026-07-14T23:30:24
updated: 2026-07-14T23:42:04
---

本文用于在 macOS 上从零安装 JDK 和 Apache Maven，并完成 `JAVA_HOME`、Zsh 与真实构建验证。内容同时覆盖 Apple Silicon 和 Intel Mac，主线提供 Eclipse Temurin 安装包与 Homebrew 两种方式。

开始前先阅读 [[Java 与 Maven 环境搭建概览]]，按项目要求选择 JDK 主版本。需要多个 JDK 时阅读 [[Java 版本管理与环境变量配置]]；Maven 仓库、代理、Wrapper 和 Toolchains 见 [[Maven 常用配置与仓库管理]]。

## 目标与完成标准

| 阶段 | 完成结果 | 关键验证 |
| --- | --- | --- |
| 确认平台 | 知道 macOS 版本和 CPU 架构 | `sw_vers`、`uname -m` |
| 安装 JDK | macOS 能发现目标 JDK | `/usr/libexec/java_home -V` |
| 配置 Java | 终端与 `JAVA_HOME` 指向预期 JDK | `java -version`、`javac -version` |
| 安装 Maven | Maven 能由预期 JDK 启动 | `mvn -version` |
| 构建验证 | Java 和 Maven 均完成一次真实构建 | `javac`、`mvn test` |

## 1. 检查 macOS、CPU 和现有环境

打开 Terminal.app 或常用终端，执行：

```bash
sw_vers
uname -m
printf 'shell=%s\n' "$SHELL"
ps -p $$ -o comm=

command -v java || true
command -v javac || true
command -v mvn || true
/usr/libexec/java_home -V 2>&1 || true
java -version 2>&1 || true
javac -version 2>&1 || true
mvn -version 2>&1 || true
```

CPU 架构通常是：

| `uname -m` 输出 | Mac 类型 | 下载时应选择 |
| --- | --- | --- |
| `arm64` | Apple Silicon，例如 M 系列 | macOS AArch64、ARM64 或 Apple Silicon 构建 |
| `x86_64` | Intel Mac，或某些在 Rosetta 下运行的终端 | macOS x64 或 x86_64 构建 |

> [!warning] 不要混装错误架构
> Apple Silicon 可以通过 Rosetta 运行部分 x86_64 程序，但本地开发应优先使用 arm64 JDK、Homebrew 和 Maven。混合架构会让终端、IDE、JNI 本地库和构建插件出现难以判断的问题。

## 2. 选择 JDK 安装路线

| 路线 | 适合场景 | 特点 |
| --- | --- | --- |
| Eclipse Temurin `.pkg` | 第一次安装、希望由 macOS 标准 JDK 目录管理 | 图形安装直观，`/usr/libexec/java_home` 可直接发现 |
| Homebrew 版本化 OpenJDK | 已经使用 Homebrew 管理开发工具 | 升级方便；版本化 Formula 是 keg-only，需要按提示注册或配置路径 |
| SDKMAN | 多项目需要频繁切换 JDK 和 Maven | 用户级安装，不替代系统或 Homebrew 包管理；详见 [[Java 版本管理与环境变量配置]] |

只选择一条主线完成首次安装。不要同时安装多个发行版后再猜测终端会选择哪一个。

## 3. 路线 A：安装 Eclipse Temurin

1. 打开 [Eclipse Temurin 发布页](https://adoptium.net/temurin/releases/)。
2. 选择项目要求的 JDK 主版本。
3. Operating System 选择 macOS。
4. Architecture 按 `uname -m` 选择 AArch64 或 x64。
5. Package Type 选择 JDK，而不是 JRE。
6. 下载 `.pkg` 安装包并完成 macOS 安装流程。

Temurin 官方 `.pkg` 会把 JDK 安装到 macOS 标准 JavaVirtualMachines 目录。完成后验证：

```bash
/usr/libexec/java_home -V
java -version
javac -version
```

如果 macOS 阻止打开安装包，应回到“系统设置 → 隐私与安全”核对开发者和文件来源，不要通过关闭 Gatekeeper 或执行来源不明的绕过命令解决。

## 4. 路线 B：使用 Homebrew 安装 OpenJDK

先确认 Homebrew 可用，并检查安装位置与版本：

```bash
brew --version
brew --prefix
brew update
brew search openjdk
```

安装项目需要的版本。例如 JDK 25：

```bash
brew install openjdk@25
```

如果项目要求 JDK 21：

```bash
brew install openjdk@21
```

版本化 OpenJDK Formula 通常是 keg-only。Homebrew 当前提示需要创建符号链接，才能让 macOS 的系统 Java 包装器和 `/usr/libexec/java_home` 发现它。JDK 25 示例：

```bash
sudo ln -sfn \
  "$(brew --prefix)/opt/openjdk@25/libexec/openjdk.jdk" \
  /Library/Java/JavaVirtualMachines/openjdk-25.jdk
```

JDK 21 示例：

```bash
sudo ln -sfn \
  "$(brew --prefix)/opt/openjdk@21/libexec/openjdk.jdk" \
  /Library/Java/JavaVirtualMachines/openjdk-21.jdk
```

不要同时执行两条后就假定默认版本已经切换。先列出所有 JDK：

```bash
/usr/libexec/java_home -V
```

Homebrew 的 Formula 提示可能随版本变化。安装后应再运行 `brew info openjdk@25` 或对应 Formula 名称，以当前 Caveats 为准。

## 5. 配置 `JAVA_HOME` 与 Zsh

macOS 提供 `/usr/libexec/java_home`，可以按主版本查找已注册的 JDK。先测试项目目标版本，例如 JDK 25：

```bash
/usr/libexec/java_home -v 25
"$(/usr/libexec/java_home -v 25)/bin/java" -version
"$(/usr/libexec/java_home -v 25)/bin/javac" -version
```

现代 macOS 新账户通常使用 Zsh。把目标版本写入 `~/.zshrc`：

```bash
printf '%s\n' \
  'export JAVA_HOME="$(/usr/libexec/java_home -v 25)"' \
  'export PATH="$JAVA_HOME/bin:$PATH"' \
  >> "$HOME/.zshrc"

source "$HOME/.zshrc"
```

如果项目使用 JDK 21，把两处 `25` 改为 `21`。配置前应先搜索已有定义，避免重复追加：

```bash
grep -nE 'JAVA_HOME|openjdk|JavaVirtualMachines' \
  "$HOME/.zshrc" "$HOME/.zprofile" 2>/dev/null || true
```

验证当前选择：

```bash
printf 'JAVA_HOME=%s\n' "$JAVA_HOME"
test -x "$JAVA_HOME/bin/java"
test -x "$JAVA_HOME/bin/javac"
java -version
javac -version
```

`~/.zprofile` 与 `~/.zshrc` 的加载场景不同。日常交互终端配置应保持清晰，具体边界可结合 [[macOS 从零配置 Oh My Zsh]] 阅读。需要按项目切换版本时，不应反复编辑这两个文件，改用 [[Java 版本管理与环境变量配置]] 中的 SDKMAN 或 Toolchains。

## 6. 安装 Maven

### 路线 A：Homebrew

Apache Maven 官方安装页支持 Homebrew：

```bash
brew install maven
mvn -version
```

截至 2026-07-14，Homebrew `maven` Formula 提供 Maven 3.9.16，但它也依赖 Homebrew 当前的非版本化 `openjdk`。这可能在已有版本化 JDK 之外再安装一个较新的 JDK。安装完成后必须查看 `mvn -version` 的 `Java home`，确认 Maven 没有意外切换项目 JDK。

Homebrew 安装或升级后可查看来源：

```bash
brew info maven
type -a mvn
mvn -version
```

### 路线 B：SDKMAN

已经使用 SDKMAN 管理 Java 时，也可以让它管理 Maven：

```bash
sdk list maven
sdk install maven
mvn -version
```

SDKMAN 的安装、精确版本选择和 `.sdkmanrc` 用法见 [[Java 版本管理与环境变量配置]]。

### 路线 C：项目 Maven Wrapper

仓库存在 `mvnw` 时，项目构建优先使用：

```bash
./mvnw -version
./mvnw test
```

Wrapper 会按项目配置下载并使用指定 Maven 版本，因此开发者不必让全局 Maven 与项目完全一致。第一次下载仍需要网络，并应校验项目提交的 Wrapper 配置；详见 [[Maven 常用配置与仓库管理]]。

## 7. 编译并运行最小 Java 程序

```bash
work_dir="$(mktemp -d)"
cd "$work_dir"

printf '%s\n' \
  'public class HelloJava {' \
  '    public static void main(String[] args) {' \
  '        System.out.println("Java environment works on macOS");' \
  '    }' \
  '}' \
  > HelloJava.java

javac HelloJava.java
java HelloJava
```

预期输出：

```text
Java environment works on macOS
```

检查并清理：

```bash
file "$(command -v java)"
ls -l HelloJava.java HelloJava.class
cd -
rm -rf "$work_dir"
```

## 8. 完成一次最小 Maven 构建

已有项目优先运行其 Wrapper。没有项目时，可临时生成示例项目：

```bash
work_dir="$(mktemp -d)"
cd "$work_dir"

mvn -B archetype:generate \
  -DgroupId=dev.example \
  -DartifactId=java-env-check \
  -DarchetypeArtifactId=maven-archetype-quickstart \
  -DarchetypeVersion=1.5

cd java-env-check
mvn test
```

看到 `BUILD SUCCESS` 后，再确认 Maven 实际环境并清理：

```bash
mvn -version
mvn help:effective-settings
cd /
rm -rf "$work_dir"
```

## 9. IDE 中的 JDK

终端配置不会自动保证 IDE 使用同一个 JDK。至少检查：

- Project SDK 或 Project JDK。
- Module SDK。
- Maven Runner JDK。
- Maven Importer JDK。
- IDE 内置终端启动的 Shell 与 `JAVA_HOME`。

IDE 可能由图形界面启动，不一定读取与 Terminal.app 相同的 Shell 启动文件。最可靠的判断是同时比较 IDE 设置、IDE 内置终端中的 `mvn -version`，以及外部终端中的 `mvn -version`。

## 10. 升级与卸载

Homebrew 管理的工具使用：

```bash
brew outdated
brew upgrade openjdk@25 maven
```

升级前先运行项目测试。卸载时明确指定 Formula：

```bash
brew uninstall maven
brew uninstall openjdk@25
```

如果为 Homebrew JDK 创建过 `/Library/Java/JavaVirtualMachines/openjdk-25.jdk` 符号链接，应先用 `ls -l` 确认它确实指向 Homebrew，再删除该符号链接。不要递归删除整个 `/Library/Java/JavaVirtualMachines`。

Temurin `.pkg` 的卸载应按 Adoptium 当前文档确认对应目录和安装收据；不要仅删除 `java` 命令。卸载任何 JDK 前，先运行 `/usr/libexec/java_home -V` 并确认没有项目和 IDE 依赖该版本。

## 中国大陆网络环境提示

JDK 安装包、Homebrew Bottle、Apache Maven 和 Maven Central 是不同下载路径。分别检查：

```bash
curl -I https://adoptium.net/
curl -I https://formulae.brew.sh/
curl -I https://maven.apache.org/
curl -I https://repo.maven.apache.org/maven2/
```

企业设备应使用组织批准的代理、证书和 Maven 仓库管理器。不要把来源不明的 Homebrew 安装脚本、JDK 二次打包或公共 Maven 镜像写入共享配置。Maven 网络配置见 [[Maven 常用配置与仓库管理]]。

## 常见问题

### `/usr/libexec/java_home -V` 看不到 Homebrew JDK

先执行：

```bash
brew info openjdk@25
ls -la /Library/Java/JavaVirtualMachines/
```

版本化 Homebrew Formula 通常需要按 Caveats 创建符号链接。不要猜路径，应使用 `brew --prefix` 和 `brew info` 提供的当前路径。

### `java` 弹出“无法找到 Java Runtime”

说明 macOS 系统包装器没有找到已注册的 JDK，或者当前 Shell 的 `PATH` 指向无效目录。检查：

```bash
/usr/libexec/java_home -V
type -a java
printf 'JAVA_HOME=%s\n' "${JAVA_HOME:-<未设置>}"
```

### Homebrew 安装 Maven 后 Java 版本变化

查看所有 JDK、命令路径和 Maven 使用的 Java：

```bash
/usr/libexec/java_home -V
type -a java
type -a mvn
mvn -version
```

用明确的 `JAVA_HOME="$(/usr/libexec/java_home -v 25)"` 选择项目 JDK，或把示例中的 `25` 换成项目要求的主版本；也可以改用 SDKMAN、Maven Toolchains。完整步骤见 [[Java 与 Maven 环境排障与维护]]。

### 修改 `~/.zshrc` 后没有生效

打开新终端，或者执行：

```bash
source "$HOME/.zshrc"
rehash
```

再用 `type -a java` 和 `mvn -version` 判断，而不是只看 `echo "$JAVA_HOME"`。

## 官方参考资料

- [Eclipse Temurin：macOS PKG 安装](https://adoptium.net/installation/macOS/)
- [Eclipse Temurin 发布下载](https://adoptium.net/temurin/releases/)
- [Homebrew：安装](https://brew.sh/)
- [Homebrew Formula：OpenJDK 25](https://formulae.brew.sh/formula/openjdk@25)
- [Homebrew Formula：OpenJDK 21](https://formulae.brew.sh/formula/openjdk@21)
- [Homebrew Formula：Maven](https://formulae.brew.sh/formula/maven)
- [Apache Maven：安装](https://maven.apache.org/install.html)
