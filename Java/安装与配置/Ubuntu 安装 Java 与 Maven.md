---
title: Ubuntu 安装 Java 与 Maven
aliases:
  - Ubuntu 安装 JDK 与 Maven
  - Linux 安装 Java 与 Maven
tags:
  - Java
  - Java/安装
  - Java/Linux
  - Java/Ubuntu
  - Maven
created: 2026-07-14T23:30:24
updated: 2026-07-14T23:42:04
---

本文用于在 Ubuntu 上从零安装 JDK 和 Apache Maven，并完成环境变量与真实构建验证。主线采用 Ubuntu 官方 apt 仓库，适合个人开发机、虚拟机以及允许使用发行版软件包的服务器。

开始前先阅读 [[Java 与 Maven 环境搭建概览]]，确认项目需要的 JDK 主版本。需要同时维护多个 JDK 时，再阅读 [[Java 版本管理与环境变量配置]]；Maven 仓库、代理、Wrapper 和 Toolchains 配置见 [[Maven 常用配置与仓库管理]]。

## 目标与完成标准

| 阶段 | 完成结果 | 验证命令 |
| --- | --- | --- |
| 安装 JDK | `java` 与 `javac` 都来自预期 JDK | `java -version`、`javac -version` |
| 确认环境 | `JAVA_HOME` 指向 JDK 根目录 | `printf '%s\n' "$JAVA_HOME"` |
| 安装 Maven | Maven 能由预期 JDK 启动 | `mvn -version` |
| Java 验证 | 能编译并运行最小程序 | `javac`、`java` |
| Maven 验证 | 能解析插件并完成测试 | `mvn test` 或 `./mvnw test` |

> [!important] 项目要求优先于系统默认
> `default-jdk` 会跟随 Ubuntu 发行版选择默认 JDK，适合没有版本约束的学习环境。已有项目如果明确要求 JDK 17、21 或 25，应安装对应版本，并在升级前确认框架、插件、CI 和生产环境兼容性。

## 1. 检查系统与现有环境

先记录 Ubuntu 版本、CPU 架构、当前 Shell 和已有命令：

```bash
. /etc/os-release
printf 'system=%s\n' "$PRETTY_NAME"
printf 'codename=%s\n' "$VERSION_CODENAME"
printf 'arch=%s\n' "$(dpkg --print-architecture)"
printf 'shell=%s\n' "$SHELL"

command -v java || true
command -v javac || true
command -v mvn || true
java -version 2>&1 || true
javac -version 2>&1 || true
mvn -version 2>&1 || true
```

如果机器已经承载 Java 服务、CI Runner 或其他用户的构建任务，不要直接删除旧 JDK。先记录服务使用的 Java 路径、systemd unit、容器镜像和构建配置，再安排切换。

## 2. 选择并安装 JDK

### 路线 A：安装 Ubuntu 默认 JDK

没有既有项目约束时，可以让 Ubuntu 选择该发行版的默认开发工具包：

```bash
sudo apt update
sudo apt install -y default-jdk
```

`default-jdk` 是一个随 Ubuntu 发行版维护的元包。它的优点是安装和安全更新简单，缺点是不同 Ubuntu 版本可能指向不同 JDK 主版本，因此团队不能只写“安装 `default-jdk`”而不记录实际版本。

安装后立即确认：

```bash
java -version
javac -version
```

### 路线 B：安装项目指定的 OpenJDK

先查看当前 Ubuntu 仓库提供哪些 JDK 包：

```bash
apt-cache search '^openjdk-[0-9]+-jdk$'
apt-cache policy openjdk-21-jdk openjdk-25-jdk
```

然后安装项目明确要求且当前仓库确实提供的版本。例如项目要求 JDK 21：

```bash
sudo apt update
sudo apt install -y openjdk-21-jdk
```

如果 `apt-cache policy` 没有 Candidate，不要把其他 Ubuntu 版本的软件源硬加到当前系统。可以选择受支持的 Ubuntu 版本、Eclipse Temurin 官方 DEB 仓库、SDKMAN 或组织提供的软件仓库；多版本路线见 [[Java 版本管理与环境变量配置]]。

> [!warning] 不要只安装 `-jre` 包
> 后端开发、Maven 编译和测试需要 `javac` 等工具，应安装 `openjdk-<版本>-jdk`。只有明确只负责运行已构建程序的机器，才可能选择更精简的运行时。

## 3. 确认 Java 路径与 `JAVA_HOME`

查看命令实际指向的文件：

```bash
type -a java
type -a javac
readlink -f "$(command -v java)"
readlink -f "$(command -v javac)"
```

可以从 `javac` 的真实路径计算当前 JDK 根目录：

```bash
jdk_home="$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")"
printf 'detected JDK home: %s\n' "$jdk_home"
"$jdk_home/bin/java" -version
"$jdk_home/bin/javac" -version
```

如果 Maven、应用脚本或 IDE 明确需要 `JAVA_HOME`，为当前用户配置它。先确认当前 Shell：

```bash
ps -p $$ -o comm=
```

Bash 交互终端通常使用 `~/.bashrc`，Zsh 交互终端通常使用 `~/.zshrc`。下面以 Bash 为例，并使用当前安装的 `javac` 动态确定路径：

```bash
printf '%s\n' \
  'export JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v javac)")")")"' \
  'export PATH="$JAVA_HOME/bin:$PATH"' \
  >> "$HOME/.bashrc"

source "$HOME/.bashrc"
```

如果当前使用 Zsh，把目标文件改为 `~/.zshrc`。Shell 配置的职责和加载顺序可结合 [[Ubuntu 从零安装 Oh My Zsh]] 理解。

重新验证：

```bash
printf 'JAVA_HOME=%s\n' "$JAVA_HOME"
test -x "$JAVA_HOME/bin/java"
test -x "$JAVA_HOME/bin/javac"
java -version
javac -version
```

> [!note] 并非所有场景都必须手工设置 `JAVA_HOME`
> 如果 `java` 已在 `PATH` 中，Maven 也可以启动。设置 `JAVA_HOME` 的价值在于让 Maven、IDE、应用脚本和其他工具明确选择同一个 JDK。存在多个 JDK 时，不要反复向启动文件追加多组互相覆盖的配置。

## 4. 安装 Maven

### 路线 A：使用 Ubuntu apt

这是维护成本最低的路线：

```bash
sudo apt update
sudo apt install -y maven
```

验证 Maven 自身版本以及它实际使用的 Java：

```bash
mvn -version
```

重点检查输出中的：

- `Apache Maven`：Maven 版本。
- `Maven home`：Maven 安装目录。
- `Java version`：Maven 进程正在使用的 JDK 版本。
- `Java home`：Maven 进程实际使用的 JDK 路径。
- `OS name` 和 `arch`：操作系统与 CPU 架构。

Ubuntu 仓库中的 Maven 版本可能落后于 Apache 当前稳定版，但会由发行版维护。已有项目如果带有 `mvnw`，优先使用项目 Wrapper；需要精确版本时选择 SDKMAN 或官方二进制归档。

### 路线 B：安装 Apache Maven 官方二进制归档

截至 2026-07-14，Apache Maven 当前稳定版是 3.9.16。执行前先查看 [Apache Maven 下载页](https://maven.apache.org/download.cgi)，把变量改为当时需要的稳定版本，不要照抄已经过期的版本号。

```bash
MAVEN_VERSION=3.9.16
cd /tmp

curl -fLO "https://dlcdn.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz"
curl -fLO "https://downloads.apache.org/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz.sha512"
sha512sum -c "apache-maven-${MAVEN_VERSION}-bin.tar.gz.sha512"
```

只有校验成功后才解压：

```bash
sudo tar -xzf "apache-maven-${MAVEN_VERSION}-bin.tar.gz" -C /opt
sudo ln -sfn "/opt/apache-maven-${MAVEN_VERSION}" /opt/maven
```

将 Maven 加入当前 Shell。Bash 使用 `~/.bashrc`，Zsh 使用 `~/.zshrc`：

```bash
printf '%s\n' \
  'export MAVEN_HOME=/opt/maven' \
  'export PATH="$MAVEN_HOME/bin:$PATH"' \
  >> "$HOME/.bashrc"

source "$HOME/.bashrc"
mvn -version
```

现代 Maven 不要求设置 `M2_HOME`；`MAVEN_HOME` 也不是启动 Maven 的必要条件，只要 Maven 的 `bin` 目录在 `PATH` 中即可。这里保留 `MAVEN_HOME` 是为了让安装位置更明确。

> [!important] 不要混用 apt Maven 与 `/opt/maven`
> 如果两条路线都安装过，`PATH` 中靠前的目录决定实际执行哪一个 `mvn`。使用 `type -a mvn` 和 `mvn -version` 确认结果，不要根据“最近安装的是谁”来猜。

## 5. 编译并运行最小 Java 程序

创建临时目录和源文件：

```bash
work_dir="$(mktemp -d)"
cd "$work_dir"

printf '%s\n' \
  'public class HelloJava {' \
  '    public static void main(String[] args) {' \
  '        System.out.println("Java environment works");' \
  '    }' \
  '}' \
  > HelloJava.java

javac HelloJava.java
java HelloJava
```

预期输出：

```text
Java environment works
```

检查生成的字节码文件后清理临时目录：

```bash
ls -l HelloJava.java HelloJava.class
cd -
rm -rf "$work_dir"
```

## 6. 完成一次最小 Maven 构建

如果已有 Maven 项目，优先在项目根目录执行 Wrapper：

```bash
./mvnw -version
./mvnw test
```

没有现成项目时，可以用 Maven Archetype 在临时目录生成示例项目。这一步需要访问 Maven Central 或组织配置的仓库：

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

成功后应看到 `BUILD SUCCESS`。再查看生效配置，并清理临时目录：

```bash
mvn help:effective-settings
cd /
rm -rf "$work_dir"
```

如果依赖下载失败，不要立刻更换 JDK；先按 [[Java 与 Maven 环境排障与维护]] 区分 DNS、代理、证书、认证、镜像和本地仓库问题。

## 7. 多 JDK 管理

Ubuntu 安装多个 JDK 后，可以查看候选项：

```bash
update-alternatives --list java
update-alternatives --list javac
```

切换系统命令时，`java` 与 `javac` 必须一起核对：

```bash
sudo update-alternatives --config java
sudo update-alternatives --config javac
```

完成后重新计算 `JAVA_HOME` 并运行 `mvn -version`。系统级 alternatives、SDKMAN、Maven Toolchains 和项目级版本管理的边界详见 [[Java 版本管理与环境变量配置]]。

## 8. 升级与卸载

apt 安装的 JDK 和 Maven 由系统包管理器升级：

```bash
sudo apt update
apt list --upgradable 2>/dev/null | grep -E 'openjdk|maven' || true
sudo apt upgrade
```

升级前应先在项目中运行测试。需要卸载某个明确版本时，先确认没有服务、IDE 或 CI 任务依赖它，再执行类似命令：

```bash
sudo apt remove openjdk-21-jdk
sudo apt remove maven
sudo apt autoremove
```

不要为了卸载工具顺手删除 `~/.m2/repository`。其中是可重新下载的依赖缓存，但删除会造成下一次构建重新下载全部依赖；`~/.m2/settings.xml` 还可能包含企业仓库和代理配置，应先备份。

手工安装到 `/opt` 的 Maven 应先从 Shell 启动文件中移除对应 `MAVEN_HOME` 和 `PATH` 配置，再删除明确的版本目录和 `/opt/maven` 符号链接。不要使用模糊通配符删除 `/opt` 中的其他工具。

## 其他 Linux 发行版

本文命令以 Ubuntu 为主。不同发行版的 JDK 包名、仓库、服务策略和默认版本可能不同：

| 发行版家族 | 常用包管理器 | Maven 官方安装页给出的命令入口 |
| --- | --- | --- |
| Debian、Ubuntu | apt | `sudo apt install maven` |
| Fedora | dnf | `sudo dnf install maven` |
| RHEL 或旧版兼容环境 | dnf 或 yum | `sudo dnf install maven` 或 `sudo yum install maven` |

表中的 Maven 命令只说明包管理器入口，不代表每个发行版都提供同一 Maven 或 JDK 版本。非 Ubuntu 系统应改查该发行版的官方文档，不要直接复制 Ubuntu 软件源和包名。

## 中国大陆网络环境提示

> [!tip] 分层检查网络路径
> apt、JDK 下载、Apache Maven 下载和 Maven Central 依赖解析是不同网络路径。浏览器能打开网页，不代表 apt、`curl` 或 Maven JVM 进程已正确使用代理。

先分别检查：

```bash
curl -I https://ubuntu.com/
curl -I https://maven.apache.org/
curl -I https://repo.maven.apache.org/maven2/
```

企业网络应使用组织提供的代理、证书和 Maven 仓库管理器。不要把来源不明、可能失效的第三方镜像地址写进共享配置；Maven 的代理和镜像配置见 [[Maven 常用配置与仓库管理]]。

## 常见问题

### `java` 成功但 `javac` 不存在

通常只安装了 JRE 或精简运行时。检查已安装包：

```bash
dpkg -l | grep -E 'openjdk|default-jdk|default-jre'
```

按项目要求安装对应的 `-jdk` 包。

### 修改配置后版本没有变化

先开一个全新终端，再检查命令缓存和所有候选路径：

```bash
hash -r
type -a java
type -a javac
type -a mvn
```

如果使用 Zsh，可以执行 `rehash`。仍有问题时参见 [[Java 与 Maven 环境排障与维护]]。

### `mvn -version` 使用了另一个 JDK

比较以下输出：

```bash
printf 'JAVA_HOME=%s\n' "${JAVA_HOME:-<未设置>}"
readlink -f "$(command -v java)"
readlink -f "$(command -v javac)"
mvn -version
```

确认 `JAVA_HOME`、`PATH`、alternatives 和 Maven 启动脚本没有指向不同 JDK。

## 官方参考资料

- [Ubuntu：配置 Java 开发环境](https://ubuntu.com/developers/docs/howto/java-setup/)
- [Ubuntu Packages](https://packages.ubuntu.com/)
- [Eclipse Adoptium：Linux 安装包](https://adoptium.net/installation/linux/)
- [Apache Maven：安装](https://maven.apache.org/install.html)
- [Apache Maven：下载与校验](https://maven.apache.org/download.cgi)
- [Apache Maven Wrapper](https://maven.apache.org/tools/wrapper/)
