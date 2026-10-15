# Docker 应用组管理器（Docker Group Manager）

Docker 应用组管理器是一款面向 NAS 和其他 Docker 主机的容器分组管理工具。可以将有关联的多个容器保存为一个应用组，集中查看并执行启动、停止、重启操作，方便管理由多个容器组成的应用。

**当前版本：** `1.5.1`  
**Docker Hub 镜像：** [`jieyousuozhang/docker-group-manager`](https://hub.docker.com/r/jieyousuozhang/docker-group-manager)  
**镜像标签：** `1.5.1`  
**支持架构：** `linux/amd64`（x86-64）

> 本仓库用于介绍、部署和发布应用，不包含应用源码。应用镜像已发布至 Docker Hub，安装时直接拉取，无需下载或导入 `.tar` 镜像文件。

## 主要功能

- **容器分组：** 读取本机 Docker 容器列表，将一个主容器和多个辅助容器保存为应用组。
- **一键操作：** 对应用组执行启动、停止和重启操作。
- **持久化保存：** 通过 SQLite 保存应用组关系。
- **搜索与排序：** 支持实时搜索，以及名称 A→Z、名称 Z→A、运行中优先和已停止优先排序。
- **成员扫描：** 扫描并同步容器成员变化，同时保留用户手动修改的显示名称。

## 安装前提

- Docker 20.10 或更新版本；推荐使用 Docker Compose 插件。
- Docker 主机为 **x86-64（amd64）** 架构；当前镜像未提供 ARM64 版本。
- 主机能够访问 Docker Hub 并拉取镜像。
- 能够挂载本机 Docker Socket：`/var/run/docker.sock`。
- 建议仅在可信局域网环境使用。

## 安装方法一：Docker Compose（推荐）

无需下载镜像压缩包，Docker Compose 会从 Docker Hub 拉取镜像。

**1. 创建应用目录**

在 NAS 或 Docker 主机终端执行：

```bash
mkdir -p docker-group-manager/data
cd docker-group-manager
```

**2. 创建 `docker-compose.yml`**

在当前目录创建该文件，并填入以下内容：

```yaml
services:
  docker-group-manager:
    image: jieyousuozhang/docker-group-manager:1.5.1
    container_name: docker-group-manager
    restart: unless-stopped
    ports:
      - "18080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
    environment:
      DATABASE_PATH: /data/app.db
      SELF_CONTAINER_NAME: docker-group-manager
```

如果从 GitHub 下载了本仓库提供的 `docker-compose.yml`，也可以直接使用，但请确认其中的 `image:` 已指向上述 Docker Hub 镜像。

**3. 拉取并启动**

在 `docker-compose.yml` 所在目录执行：

```bash
docker compose pull
docker compose up -d
```

首次安装时会自动拉取镜像，无需执行 `docker load`。之后访问：

```text
http://NAS_IP:18080
```

将 `NAS_IP` 替换为 NAS 或 Docker 主机的局域网 IP 地址。

## 安装方法二：Docker 命令启动

没有 Docker Compose 插件时，可以直接运行：

```bash
mkdir -p docker-group-manager/data
cd docker-group-manager

docker pull jieyousuozhang/docker-group-manager:1.5.1

docker run -d \
  --name docker-group-manager \
  --restart unless-stopped \
  -p 18080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$(pwd)/data:/data" \
  -e DATABASE_PATH=/data/app.db \
  -e SELF_CONTAINER_NAME=docker-group-manager \
  jieyousuozhang/docker-group-manager:1.5.1
```

启动后同样通过 `http://NAS_IP:18080` 访问。

## 运行检查

在安装应用的 Docker 主机上执行：

```bash
docker ps --filter name=docker-group-manager
curl http://127.0.0.1:18080/api/health
```

正常情况下，健康检查响应应包含如下信息：

```json
{"ok":true,"docker":true,"version":"1.5.1"}
```

如果无法访问，请检查容器是否正常运行、端口是否被占用，以及 NAS 防火墙或网络访问规则。

## 数据持久化

默认将部署目录下的 `./data` 映射到容器内 `/data`，其中：

- `./data/app.db`：应用组配置数据库。

重新拉取镜像、升级应用或重建容器时，只要保留原有数据目录并继续正确挂载，已保存的应用组配置就可以继续使用。升级前建议备份 `./data`。

## 端口修改

默认端口映射为宿主机 `18080` → 容器 `8080`。

如需将宿主机访问端口改为 `19090`，把 Compose 配置修改为：

```yaml
ports:
  - "19090:8080"
```

此时通过 `http://NAS_IP:19090` 访问。冒号左侧为宿主机端口，右侧为容器端口；仅修改左侧即可。

## 更新版本

使用 Compose 安装时，在部署目录执行：

```bash
docker compose pull
docker compose up -d
```

以上命令会检查并拉取 Compose 文件所指定标签的镜像，然后按需重建容器。**如果今后发布新版本，还需要将 `docker-compose.yml` 中的镜像标签从 `1.5.1` 改为对应的新版本号。**

更新前建议备份 `./data`。不要为了更新镜像而删除数据目录。

## 卸载

使用 Compose 安装时：

```bash
docker compose down
```

使用 `docker run` 安装时：

```bash
docker stop docker-group-manager
docker rm docker-group-manager
```

上述操作不会主动删除部署目录中的 `./data`。如需清除全部应用组配置，请在确认已备份后手动删除该目录。

## 安全须知

本工具通过 Docker Socket 管理 Docker 容器。访问 Docker Socket 通常意味着拥有对 Docker 主机的高权限，因此请注意：

- 仅部署在可信设备上，优先通过局域网访问。
- **不要将管理页面的访问端口直接暴露到公网。** 如确需远程访问，应配置可靠的访问控制、身份认证和 HTTPS。
- 不要将未知来源的容器加入应用组。
- 不要将管理器自身加入应用组。
- 删除应用组仅删除管理器保存的分组记录，不会删除对应的 Docker 容器。

## 镜像信息

- **镜像仓库：** `docker.io/jieyousuozhang/docker-group-manager`
- **当前版本：** `1.5.1`
- **平台：** `linux/amd64`
- **镜像清单摘要（Digest）：** `sha256:d0d09078229bc20bdd4c5538805b4233e5c06e53249ad8d1d9cfdb87b0a0a9c9`

Docker Hub 页面：<https://hub.docker.com/r/jieyousuozhang/docker-group-manager>

镜像已发布至 Docker Hub，用户无需自行构建镜像，也无需手动上传镜像。
