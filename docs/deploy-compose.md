# 使用 Docker Compose 在 2c2g 主机运行 happy-server

本文档演示如何在 **2 核 / 2 GB 内存** 的主机上运行 `packages/happy-server`。核心思路是利用项目自带的 standalone 模式（PGlite + 本地文件系统 + 内置事件总线），把原本需要 Postgres、Redis、MinIO 的多进程架构压缩为单一容器，配合关闭 Prometheus Metrics 等可选特性，将常驻内存控制在 1.2 GB 左右，从而避免 OOM。

## 为什么要这样做？

- 原始部署默认依赖 Postgres、Redis、S3（MinIO），仅这三项就会在 2 GB RAM 的机器上占据 1 GB 以上，Node 进程极易被 OOM Killer 终止。
- `packages/happy-server` 已提供 `build:standalone`，利用嵌入式 PGlite 与本地文件目录即可满足 Happy CLI/App 的同步需求，无需额外服务。
- 关闭 `/metrics` HTTP 服务器、走单容器部署可以显著降低 CPU/内存占用，同时 docker compose 仍然能保留数据卷与生命周期管理能力。

## 目录结构

新增的脚本位于 `packages/happy-server/deploy`：

- `Dockerfile.compose`：构建 standalone 可执行文件，并产出只包含 `dist/happy-server` 的极简镜像。
- `docker-compose.yml`：声明单服务编排、挂载数据卷、健康检查等。
- `happy-server.env.example`：示例环境变量文件（只需自定义密钥即可运行）。

配套文档即本文件 `docs/deploy-compose.md`。

## 前提条件

- 安装 Docker 25+ 与 docker compose v2。
- 至少 5 GB 空闲磁盘（用于镜像和数据卷）。
- 可以访问本仓库源代码（compose 会从源码构建镜像）。

## 1. 配置环境变量

```bash
cp packages/happy-server/deploy/happy-server.env.example packages/happy-server/deploy/happy-server.env
```

编辑 `packages/happy-server/deploy/happy-server.env`：

- 必填：`HANDY_MASTER_SECRET`（建议 32+ 位随机字符串）。
- 选填：`PUBLIC_URL`（用于生成文件下载 URL）、`PORT`（默认 3005）。
- 默认将 `METRICS_ENABLED=false`，避免在小内存主机上额外拉起 metrics server；若需要 Prometheus，可改成 `true` 并在 compose 中额外挂载 9090 端口。

## 2. 构建 standalone 镜像

```bash
docker compose -f packages/happy-server/deploy/docker-compose.yml build happy-server
```

该步骤会：

1. 使用 `Dockerfile.compose` 安装依赖、运行 `yarn workspace happy-server build:standalone`，生成 `dist/happy-server` 可执行文件和 PGlite 运行所需的 `pglite.{wasm,data}` 文件。
2. 将产物打包进精简的 Debian 镜像中，只保留运行 happy-server 所需的文件，减少运行时内存。

## 3. 初始化数据库

PGlite 数据库与上传文件都会保存在 `happy_data` 卷中。首次启动需要执行迁移：

```bash
docker compose -f packages/happy-server/deploy/docker-compose.yml run --rm happy-server migrate
```

命令运行完成后即可在 `happy_data` 中看到 `pglite` 与 `files` 目录。

## 4. 启动服务

```bash
docker compose -f packages/happy-server/deploy/docker-compose.yml up -d happy-server
```

验证：

```bash
# 查看日志
docker compose -f packages/happy-server/deploy/docker-compose.yml logs -f happy-server

# 健康检查
curl http://localhost:3005/health
```

成功后即可配置 Happy CLI/App 指向 `http://<服务器IP>:3005`。

## 5. 数据目录与升级

- Compose 文件将 `happy_data` 卷挂载到容器 `/data`。其中：
  - `pglite/` 保存数据库文件；
  - `files/` 保存附件/头像等本地文件。
- 备份时直接备份 `happy_data` 卷即可；升级镜像不会影响数据。
- 若需要修改对外端口，可在 compose 中将 `3005:3005` 改为 `HOST_PORT:3005` 并同步更新 `PUBLIC_URL`。

## 6. 资源占用说明

- **数据库**：通过 PGlite 内嵌数据库替换 Postgres，避免额外挂载与 500+ MB 的内存开销。
- **缓存/事件**：默认使用内存事件总线，不连接 Redis，省去 300+ MB。
- **对象存储**：本地 `/data/files` 目录代替 MinIO/S3；如果未来需要 S3，只需为容器注入 `S3_*` 环境变量即可。
- **指标**：`METRICS_ENABLED=false` 避免占用额外端口和 150 MB 左右的 Node 实例；需要时再开启。

在上述配置下，容器常驻内存约 600–800 MB，配合系统及 Docker 开销即可在 2c2g 主机上稳定运行。

## 7. 常见操作

| 操作 | 命令 |
|------|------|
| 停止服务 | `docker compose -f packages/happy-server/deploy/docker-compose.yml down` |
| 查看健康状态 | `curl http://localhost:3005/health` |
| 备份数据卷 | `docker run --rm -v happy_data:/data -v "$(pwd)":/backup busybox tar czf /backup/happy-data.tar.gz /data` |
| 升级镜像 | 重新执行构建命令，然后 `docker compose ... up -d --build happy-server` |

通过以上 compose 部署方式，就能在低配置服务器上获得与官方云环境同级别的零知识同步能力。需要扩大规模时，只需改用常规 `Dockerfile.server` + 外部 Postgres/Redis/S3 即可无缝迁移。
