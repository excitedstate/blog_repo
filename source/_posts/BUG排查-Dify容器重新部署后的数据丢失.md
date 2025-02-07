---
title: BUG排查-Dify容器重新部署后的数据丢失
date: 2025-02-07 19:57:29
tags:
categories:
- BUG排查
---

# BUG 解决记录

> 摘要: 由于用户环境变量中包含`PGDATA`环境变量, 且被`docker-compose`在容器部署时引用，因此未能将容器内目录映射到主机的正确位置，从而在`docker-compose down`之后`Postgre`数据丢失, 且无法在重新`up`后恢复用户数据. 另外，作者提醒读者注意, 该BUG出现在`Mac OS 15.3`, 可能无法在主流的`LINUX`发行版上复现.

## 现象描述

[Dify](https://dify.ai/zh)是开源的 LLM 应用开发平台。提供从 Agent 构建到 AI workflow 编排、RAG 检索、模型管理等能力，轻松构建和运营生成式 AI 原生应用。

Dify仍处于快速更新和迭代的阶段，升级操作十分频繁。在容器化部署场景下，我们可以使用这样一个脚本快速升级部署在本机上的`Dify`版本(这个脚本来自Dify官方).

```shell
#!/bin/bash
cd dify/docker
docker compose down
git pull origin main
docker compose pull
docker compose up -d
```

然而, 在升级重启后，我们发现，Dify自动跳转到`http://127.0.0.1/install`界面要求我们重新设置管理员密码, 而检查`dify/docker/volumes/db/data`目录却发现空空如也. 

### 预期现象

- 应用升级重启后，我们的账号和应用数据应当是自动迁移的
- `dify/docker/volumes/db/data`不应当没有文件, 即使我们没有任何操作，也应该有默认的目录存在. 

## 排查记录

> 直接说结论, 我们本地主机上设置了`PGDATA`环境变量(`/usr/local/pgsql/data`)且和Dify官方设置的不同，而`docker-compose`引用了本地主机上的环境变量，从而导致了一个不一致状态, 官方期待的`PGDATA`是`/var/lib/postgresql/data/pgdata`, 而实际上数据落在了`/usr/local/pgsql/data`, 映射的目录是前者, 数据的实际存储目录没有映射出来, 从而导致容器重启后数据丢失.
> 注意, 本文作者使用的是`Mac OS 15.3`, 本文描述的BUG可能无法在主流的`LINUX`发行版上复现.

### 官方给出的`docker-compose`文件-数据库片段

```yaml
  db:
    image: postgres:15-alpine
    restart: always
    environment:
      PGUSER: ${PGUSER:-postgres}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-difyai123456}
      POSTGRES_DB: ${POSTGRES_DB:-dify}
      PGDATA: ${PGDATA:-/var/lib/postgresql/data/pgdata}
    command: >
      postgres -c 'max_connections=${POSTGRES_MAX_CONNECTIONS:-100}'
               -c 'shared_buffers=${POSTGRES_SHARED_BUFFERS:-128MB}'
               -c 'work_mem=${POSTGRES_WORK_MEM:-4MB}'
               -c 'maintenance_work_mem=${POSTGRES_MAINTENANCE_WORK_MEM:-64MB}'
               -c 'effective_cache_size=${POSTGRES_EFFECTIVE_CACHE_SIZE:-4096MB}'
    volumes:
      - ./volumes/db/data:/var/lib/postgresql/data/pgdata
    healthcheck:
      test: [ 'CMD', 'pg_isready' ]
      interval: 1s
      timeout: 3s
      retries: 30

```

注意在`volumes`项中, 将主机的`./volumes/db/data`映射到容器的`/var/lib/postgresql/data/pgdata`处, 但`Postgre`启动后实际存储的数据位置由环境变量`PGDATA`决定. 该变量的设定如下:

```yaml
# # 用户环境变量, 传递到docker-compose应用环境变量
# # 大约315行, 这里PGDATA引用了用户环境变量, 如果用户没有设置, 则使用/var/lib/postgresql/data/pgdata
PGDATA: ${PGDATA:-/var/lib/postgresql/data/pgdata}
```

也就是说, 如果用户设置了`PGDATA`环境环境, 那就用用户自己设定的. 糟糕的是, 如果主机上部署了PG数据库, 这个变量很可能是被设置过的.

### 修改

最简单的修改是删除主机上的`PGDATA`环境变量, 但这样的修改可能会影响后续主机上PG系统的启动. 还有一种方法是修改`docker-compose.yml`的`db`片段的`volumes`项如下:

```yaml
    volumes:
      - ./volumes/db/data:${PGDATA:-/var/lib/postgresql/data/pgdata}
```

这样, 即使用户设置了`PGDATA`环境变量, 容器内的数据目录也会映射到正确的位置, 如下所示:

```shell
$ tree ./volumes/db -L 2 
./volumes/db
└── data
    ├── PG_VERSION
    ├── base
    ├── global
    ├── pg_commit_ts
    ├── pg_dynshmem
    ├── pg_hba.conf
    ├── pg_ident.conf
    ├── pg_logical
    ├── pg_multixact
    ├── pg_notify
    ├── pg_replslot
    ├── pg_serial
    ├── pg_snapshots
    ├── pg_stat
    ├── pg_stat_tmp
    ├── pg_subtrans
    ├── pg_tblspc
    ├── pg_twophase
    ├── pg_wal
    ├── pg_xact
    ├── postgresql.auto.conf
    ├── postgresql.conf
    ├── postmaster.opts
    └── postmaster.pid

19 directories, 7 files
```

如此大功告成, 之后大多数情况都可以安心升级了.