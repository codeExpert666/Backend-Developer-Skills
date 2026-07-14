---
title: 使用 docker run 启动 MySQL
aliases:
  - docker run 运行 MySQL
  - MySQL 容器快速启动
tags:
  - Docker
  - Docker/实战
  - Docker/MySQL
  - Docker/CLI
  - MySQL
created: 2026-07-15T00:15:50
updated: 2026-07-15T00:29:12
---

本文使用 `docker run` 从零启动一个可持久化的 MySQL 容器，并完成查看状态、连接、停止、重启和删除验证。它适合学习单个容器的参数含义；项目长期使用时，建议将相同配置迁移到 [[使用 Docker Compose 编排 MySQL]]。

开始前应已完成 [[Docker 安装概览]] 中的最小验证，并能正常执行：

```bash
docker version
docker info
```

## 准备镜像与数据卷

### 拉取明确版本

```bash
docker pull mysql:8.4.10
docker image inspect mysql:8.4.10 --format '{{.RepoTags}} {{.Id}}'
```

示例使用精确补丁版本，便于重复实验。准备升级时不要只修改镜像标签，应先阅读 [[MySQL 容器备份恢复与版本升级]]。

### 创建命名卷

```bash
docker volume create mysql-dev-data
docker volume inspect mysql-dev-data
```

该卷会挂载到容器的 `/var/lib/mysql`，其中包含 MySQL 数据文件。命名卷独立于容器存在；删除容器不会自动删除它。

## 启动 MySQL 容器

```bash
docker run -d \
  --name dev-mysql \
  --publish 127.0.0.1:3306:3306 \
  --mount type=volume,source=mysql-dev-data,target=/var/lib/mysql \
  --env MYSQL_ROOT_PASSWORD='replace-with-a-root-password' \
  --env MYSQL_DATABASE='app_db' \
  --env MYSQL_USER='app_user' \
  --env MYSQL_PASSWORD='replace-with-an-app-password' \
  mysql:8.4.10
```

> [!warning] 示例密码必须替换
> 上述值只是明确的占位符。命令行参数和容器环境变量并不是生产级秘密管理方案；本地开发之外应使用受控的 Secret 机制。不要把真实密码提交到 Shell 脚本、笔记或 Git 仓库。

| 参数 | 作用 |
| --- | --- |
| `-d` | 在后台运行容器 |
| `--name dev-mysql` | 为容器提供稳定名称 |
| `--publish 127.0.0.1:3306:3306` | 只在宿主机回环地址发布 MySQL 端口 |
| `--mount ...` | 将命名卷挂载到 MySQL 数据目录 |
| `MYSQL_ROOT_PASSWORD` | 首次初始化时设置 root 密码 |
| `MYSQL_DATABASE` | 首次初始化时创建业务数据库 |
| `MYSQL_USER`、`MYSQL_PASSWORD` | 首次初始化时创建业务用户并授权到指定数据库 |

`MYSQL_USER` 和 `MYSQL_PASSWORD` 必须成对使用。业务应用应使用 `app_user`，而不是长期使用拥有最高权限的 `root`。

## 等待初始化完成

新数据卷首次启动时，MySQL 要创建数据字典、系统表、用户和业务数据库，因此容器进入运行状态不代表已经能够执行 SQL。

```bash
docker ps --filter name=dev-mysql
docker logs --follow dev-mysql
```

日志出现 MySQL 已准备接受连接的消息后，按 `Ctrl+C` 退出日志跟随；这不会停止容器。

也可以循环执行只读探测：

```bash
docker exec dev-mysql \
  mysqladmin ping -h localhost -u root -p'replace-with-a-root-password'
```

当输出 `mysqld is alive` 时，说明服务端开始响应。该检查不代表业务表迁移已经完成。

## 连接 MySQL

### 在容器内使用 MySQL 客户端

```bash
docker exec -it dev-mysql mysql -u app_user -p app_db
```

按提示输入应用用户密码。不要写成 `-p真实密码`，否则密码更容易进入命令历史和进程参数。

连接后执行：

```sql
SELECT VERSION();
SELECT DATABASE();
SELECT CURRENT_USER();
SHOW DATABASES;
```

输入 `exit` 退出 MySQL 客户端。

### 从宿主机连接

如果宿主机已安装 MySQL 客户端：

```bash
mysql \
  --host=127.0.0.1 \
  --port=3306 \
  --user=app_user \
  --password \
  app_db
```

这里必须使用宿主机发布端口。若宿主机已有 MySQL 占用 `3306`，可以删除并重建容器，将参数改成 `127.0.0.1:3307:3306`，然后从宿主机连接 `3307`。

## 写入并验证持久化数据

连接 `app_db` 后执行：

```sql
CREATE TABLE IF NOT EXISTS docker_check (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message VARCHAR(100) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO docker_check (message) VALUES ('data survives container recreation');
SELECT * FROM docker_check;
```

退出客户端，删除容器但保留命名卷：

```bash
docker stop dev-mysql
docker rm dev-mysql
docker volume ls --filter name=mysql-dev-data
```

使用与之前相同的镜像、密码和卷重新创建容器：

```bash
docker run -d \
  --name dev-mysql \
  --publish 127.0.0.1:3306:3306 \
  --mount type=volume,source=mysql-dev-data,target=/var/lib/mysql \
  mysql:8.4.10
```

> [!note] 为什么第二次没有设置初始化变量
> 该卷已经包含数据库。MySQL Official Image 会保留已有数据库，`MYSQL_ROOT_PASSWORD`、`MYSQL_DATABASE`、`MYSQL_USER` 和 `MYSQL_PASSWORD` 等首次初始化变量不会重置旧数据或旧密码。连接时仍使用第一次初始化创建的凭据。

等待服务启动后重新查询：

```bash
docker exec -it dev-mysql mysql -u app_user -p app_db
```

```sql
SELECT * FROM docker_check;
```

仍能看到记录，说明数据存在命名卷中，而不是旧容器的可写层中。更完整的存储说明见 [[MySQL 容器数据持久化]]。

## 容器生命周期命令

```bash
# 查看运行中的容器
docker ps --filter name=dev-mysql

# 查看包括已停止容器在内的状态
docker ps -a --filter name=dev-mysql

# 查看最近 100 行日志
docker logs --tail 100 dev-mysql

# 查看端口发布
docker port dev-mysql

# 查看挂载、网络和环境等配置
docker inspect dev-mysql

# 查看实时资源占用
docker stats dev-mysql

# 优雅停止
docker stop dev-mysql

# 重新启动已停止的容器
docker start dev-mysql

# 重启
docker restart dev-mysql
```

`docker stop` 会向容器主进程发送终止信号，让 MySQL 有机会正常关闭。除非容器已经无法响应，否则不要把 `docker kill` 作为常规停止方式。

## 清理环境

### 只删除容器，保留数据库

```bash
docker rm --force dev-mysql
docker volume inspect mysql-dev-data
```

以后可以将 `mysql-dev-data` 挂载给兼容版本的新容器。

### 删除容器和数据库

先完成 [[MySQL 容器备份恢复与版本升级]] 中的备份与恢复验证，再执行：

```bash
docker rm --force dev-mysql
docker volume rm mysql-dev-data
```

> [!danger] 删除卷是破坏性操作
> `docker volume rm` 会删除该卷中的 MySQL 数据。镜像、容器和卷是不同对象；重新拉取镜像不能恢复已删除的数据卷。

## 常见问题

### 容器启动后马上退出

```bash
docker ps -a --filter name=dev-mysql
docker logs dev-mysql
```

重点检查是否遗漏 root 密码、挂载目录是否可写、配置参数是否无效，以及旧数据目录是否与镜像版本兼容。

### 端口已经被占用

```bash
docker ps --format 'table {{.Names}}\t{{.Ports}}'
```

改用宿主机端口 `3307` 不会改变容器内 MySQL 仍监听 `3306` 的事实。宿主机连接 `127.0.0.1:3307`，其他同网络容器仍连接容器名和 `3306`。

### 修改环境变量后密码没有变化

初始化变量只在空数据目录首次启动时生效。旧库修改密码应连接 MySQL 后执行受控的 `ALTER USER`，而不是修改 `docker run` 参数。详见 [[MySQL 容器配置与初始化]]。

## 相关笔记

- [[使用 Docker 运行 MySQL 概览]]
- [[使用 Docker Compose 编排 MySQL]]
- [[MySQL 容器数据持久化]]
- [[MySQL 容器日常维护与故障排查]]

## 官方参考资料

- [Docker Hub：MySQL Official Image](https://hub.docker.com/_/mysql)
- [Docker：docker container run](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker：Volumes](https://docs.docker.com/engine/storage/volumes/)
- [MySQL：使用 Docker 部署 MySQL Server](https://dev.mysql.com/doc/refman/8.4/en/docker-mysql-getting-started.html)
