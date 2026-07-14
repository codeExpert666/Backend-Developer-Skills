---
title: MySQL 容器数据持久化
aliases:
  - Docker MySQL 数据卷
  - MySQL 容器存储
tags:
  - Docker
  - Docker/实战
  - Docker/MySQL
  - Docker/数据持久化
  - MySQL/存储
created: 2026-07-15T00:15:50
updated: 2026-07-15T00:29:12
---

MySQL 的数据不能只保存在容器可写层中。容器本来就应该能够被重建；真正需要长期保留的是 `/var/lib/mysql` 中的数据目录、可恢复的备份，以及与当前 MySQL 版本匹配的配置。

本文说明命名卷、bind mount 和容器可写层的区别，并提供删除容器后验证数据仍然存在的完整方法。备份与版本升级另见 [[MySQL 容器备份恢复与版本升级]]。

## MySQL 数据保存在哪里

```mermaid
flowchart LR
    A[mysql:8.4.10 镜像] --> B[MySQL 容器]
    B --> C[容器可写层]
    B --> D[/var/lib/mysql]
    D --> E[(Docker 命名卷)]
```

镜像是只读模板。容器可写层适合临时运行状态，不适合保存数据库。将命名卷挂载到 `/var/lib/mysql` 后，MySQL 对该目录的写入会进入数据卷，其生命周期不再绑定某一个容器。

## 三种存储方式

| 方式 | 示例 | 生命周期 | 建议 |
| --- | --- | --- | --- |
| 容器可写层 | 不挂载 `/var/lib/mysql` | 随容器删除而丢失 | 不用于需要保留的数据 |
| Docker 命名卷 | `mysql_data:/var/lib/mysql` | 独立于容器，由 Docker 管理 | 本地开发的默认选择 |
| Bind mount | `./data:/var/lib/mysql` | 直接使用宿主机目录 | 仅在确实需要控制路径并能管理权限时使用 |

Docker 官方建议优先使用 Volume 保存容器生成的持久数据。命名卷跨 Linux、Docker Desktop 和常规 Compose 项目更容易复用，也能避免宿主机工具无意改写 MySQL 数据文件。

## 使用命名卷

### `docker run` 示例

```bash
docker volume create mysql-dev-data

docker run -d \
  --name dev-mysql \
  --mount type=volume,source=mysql-dev-data,target=/var/lib/mysql \
  --env MYSQL_ROOT_PASSWORD='replace-with-a-root-password' \
  mysql:8.4.10
```

### Compose 示例

```yaml
services:
  db:
    image: mysql:8.4.10
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

第一次执行 `docker compose up` 时，Compose 创建项目命名卷；以后启动会复用它。

## 查看实际使用的卷

### 查看 Compose 声明

```bash
docker compose config
docker compose volumes
```

### 查看运行中容器的挂载

```bash
container_id="$(docker compose ps -q db)"

docker inspect \
  --format '{{range .Mounts}}{{println .Type .Name .Source "->" .Destination}}{{end}}' \
  "$container_id"
```

输出应包含挂载目标 `/var/lib/mysql`。如果没有，数据库可能正在写入容器可写层。

### 查看卷信息

```bash
docker volume ls
volume_name='replace-with-actual-volume-name'
docker volume inspect "$volume_name"
```

Compose 默认使用“项目名 + 卷逻辑名”生成实际卷名。例如目录名为 `shop`，逻辑卷为 `mysql_data`，实际名称可能是 `shop_mysql_data`。

> [!note] 不要依赖 Docker Desktop 内部路径
> 在 macOS 和 Windows 上，Docker Engine 与卷通常位于 Docker Desktop 管理的 Linux 虚拟化环境中。`docker volume inspect` 的 `Mountpoint` 不代表可以从宿主系统直接安全编辑。通过容器、Docker CLI 和数据库工具访问数据，而不是进入内部 VM 修改文件。

## 验证数据独立于容器

先创建测试表：

```bash
docker compose exec db mysql -u app_user -p app_db
```

```sql
CREATE TABLE IF NOT EXISTS persistence_check (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    value_text VARCHAR(100) NOT NULL
);

INSERT INTO persistence_check (value_text) VALUES ('stored in a Docker volume');
SELECT * FROM persistence_check;
```

退出客户端后删除容器和项目网络，但不删除卷：

```bash
docker compose down
docker volume ls
```

重新创建容器：

```bash
docker compose up -d
docker compose exec db mysql -u app_user -p app_db
```

```sql
SELECT * FROM persistence_check;
```

记录仍然存在，说明 Compose 重建了容器，但复用了命名卷。

## Bind mount 何时使用

Bind mount 将明确的宿主机目录直接映射到 `/var/lib/mysql`：

```yaml
services:
  db:
    image: mysql:8.4.10
    volumes:
      - ./mysql-data:/var/lib/mysql
```

它适合以下少数场景：

- 组织要求数据库文件必须位于指定磁盘或挂载点。
- 已有明确的宿主机备份、权限和监控策略。
- 管理员需要控制底层文件系统容量和挂载参数。

它也带来额外责任：

- 宿主目录必须存在并允许容器内 MySQL 用户写入。
- SELinux、AppArmor、文件共享设置或宿主机权限可能阻止启动。
- 非 Docker 进程可以直接修改文件，增加损坏风险。
- 跨 macOS、Windows 和 Linux 的文件系统行为与性能不同。
- 不能把不兼容 MySQL 版本的数据目录随意挂给新镜像。

对于普通本地开发，优先使用命名卷；初始化 SQL 和 `.cnf` 配置文件可以使用只读 bind mount，数据库数据目录则交给 Docker Volume。

## 为什么不能直接复制运行中的数据目录

MySQL 运行时会同时维护数据页、redo log、undo、二进制日志和数据字典。直接复制正在变化的数据目录，可能得到时间点不一致的文件集合。

因此应区分：

- 逻辑备份：通过 `mysqldump` 等数据库工具读取一致的数据视图。
- 物理备份：使用支持 MySQL 一致性的专用工具和流程。
- 冷备份：完整停止 MySQL 并确认干净关闭后复制受控数据目录。
- Docker 卷归档：只能在理解数据库一致性前提后作为存储层操作，不能自动等同于可靠 MySQL 备份。

本地开发最容易验证的方式是逻辑备份和恢复，详见 [[MySQL 容器备份恢复与版本升级]]。

## Compose 项目名与卷名

以下因素可能改变 Compose 项目名：

- 项目目录名称变化。
- 使用 `docker compose --project-name <name>`。
- 设置 `COMPOSE_PROJECT_NAME`。
- Compose 顶层使用 `name:`。

项目名变化后，`docker compose up` 可能创建新卷，看起来像“数据库被清空”。这时不要立即初始化或删除旧卷，先检查：

```bash
docker compose ls
docker volume ls
docker ps -a --format 'table {{.Names}}\t{{.Mounts}}'
```

若确实需要跨项目复用已存在的卷，可以显式声明外部卷：

```yaml
volumes:
  mysql_data:
    external: true
    name: shared-mysql-data
```

外部卷不会由 `docker compose down -v` 自动管理，但也要求团队自行保证名称、生命周期和环境隔离。

## 安全清理顺序

### 查看空间占用

```bash
docker system df
docker volume ls
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Mounts}}'
```

### 删除容器但保留卷

```bash
docker compose down
```

### 删除确定不再需要的卷

1. 确认卷属于哪个环境。
2. 完成备份。
3. 将备份恢复到新数据库并查询关键数据。
4. 停止使用该卷的所有容器。
5. 最后删除指定卷。

```bash
confirmed_volume_name='replace-after-verification'
docker volume rm "$confirmed_volume_name"
```

> [!danger] 不要把 `docker volume prune` 当作日常清理
> 它会删除所有当前未被容器引用的本地卷。一个停止并已删除容器、但仍需保留的 MySQL 卷可能被判定为“未使用”。优先按名称逐个确认和删除。

## 常见问题

### 删除容器后数据消失

检查旧容器是否根本没有挂载 `/var/lib/mysql`，或者新容器是否挂载了另一个卷。使用 `docker inspect` 对比挂载信息。

### 重建后出现全新的空数据库

常见原因是 Compose 项目名变化、卷逻辑名变化，或在另一个 Docker context 中启动。依次检查：

```bash
docker context show
docker compose ls
docker volume ls
```

### Bind mount 报 `Permission denied`

不要直接执行 `chmod -R 777`。先查看容器日志、宿主目录所有者、SELinux 标签、Docker Desktop 文件共享设置和镜像运行用户，再根据宿主系统制定最小权限修复。

### `down -v` 后还能恢复吗

Docker 不提供数据库级“撤销删除卷”。如果没有独立备份，只能停止继续写入并评估底层存储恢复可能性，成功率和一致性都不能保证。

## 相关笔记

- [[使用 docker run 启动 MySQL]]
- [[使用 Docker Compose 编排 MySQL]]
- [[MySQL 容器配置与初始化]]
- [[MySQL 容器备份恢复与版本升级]]
- [[MySQL 容器日常维护与故障排查]]

## 官方参考资料

- [Docker：Storage](https://docs.docker.com/engine/storage/)
- [Docker：Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Docker：Bind mounts](https://docs.docker.com/engine/storage/bind-mounts/)
- [Docker Hub：MySQL Official Image 的数据存储说明](https://hub.docker.com/_/mysql)
