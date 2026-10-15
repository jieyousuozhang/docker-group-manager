# Docker 应用组管理器 GitHub 发布包

这是一个只面向安装和发布的文件夹，不包含应用源码。

发布内容：

- `docker-group-manager-1.5.1.tar`：Docker 镜像包
- `docker-compose.yml`：安装和启动配置
- `README.md`：安装说明

## 功能

- 读取本机 Docker 容器列表
- 将一个主容器和多个辅助容器保存为应用组
- 一键启动、关闭、重启应用组
- 保存应用组关系到 SQLite
- 实时搜索列表
- 按名称 A→Z、名称 Z→A、运行中优先、已停止优先排序
- 扫描同步容器成员变化
- 扫描不会覆盖你手动修改的显示名称

## 安装要求

- Docker 20.10 或更新版本
- Docker Compose 插件可用
- 机器能访问 Docker Socket
- 本工具只建议在局域网访问

## 方式一：直接从镜像 tar 安装

在本文件夹执行：

```bash
docker load -i docker-group-manager-1.5.1.tar
mkdir -p ./data
docker compose up -d
```

打开：

```text
http://NAS_IP:18080
```

确认健康检查：

```bash
curl http://127.0.0.1:18080/api/health
```

预期返回包含：

```json
{"ok":true,"docker":true,"version":"1.5.1"}
```

## 方式二：先加载镜像，再手动启动

如果本机没有 `docker compose` 插件：

```bash
docker load -i docker-group-manager-1.5.1.tar
mkdir -p ./data
docker run -d \
  --name docker-group-manager \
  --restart unless-stopped \
  -p 18080:8080 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v "$(pwd)/data:/data" \
  -e DATABASE_PATH=/data/app.db \
  -e SELF_CONTAINER_NAME=docker-group-manager \
  docker-group-manager:1.5.1
```

## 数据位置

Compose 默认把当前文件夹下的 `./data` 挂载到容器 `/data`：

- `./data/app.db`：应用组配置数据库

只要 `./data` 文件夹还在，重新创建容器不会丢失应用组配置。

## 端口和网络

默认映射：

```text
宿主 18080 -> 容器 8080
```

如果要改端口，修改 `docker-compose.yml` 的：

```yaml
ports:
  - "8080:8080"
```

左边是宿主端口，右边是容器端口。

不要把该端口直接暴露到公网。如需远程访问，应在反向代理层增加认证和 HTTPS。

## 安全说明

本工具通过 Docker Socket 管理本机容器，Docker Socket 权限很高：

- 只在可信局域网使用
- 不要把管理器发布到公网
- 不要把未知来源的容器挂入管理器
- 不要把管理器自身加入应用组
- 删除应用组只删除管理器里的组记录，不会删除 Docker 容器

## 镜像分发说明

GitHub Release 可以直接附带本文件夹中的 `docker-group-manager-1.5.1.tar`。

其他人安装时先下载 Release 附件，再执行：

```bash
docker load -i docker-group-manager-1.5.1.tar
mkdir -p ./data
docker compose up -d
```

如果希望别人通过自己的 Docker Hub / 镜像仓库拉取，需要先推送镜像：

```bash
docker tag docker-group-manager:1.5.1 <your-dockerhub>/docker-group-manager:1.5.1
docker push <your-dockerhub>/docker-group-manager:1.5.1
```

然后把 `docker-compose.yml` 的 `image:` 改成：

```yaml
image: <your-dockerhub>/docker-group-manager:1.5.1
```

这样别人可以执行：

```bash
docker compose pull
docker compose up -d
```

## 卸载

```bash
docker compose down
```

如果要同时删除数据：

```bash
rm -rf ./data
```

注意：删除 `./data` 会删除已保存的应用组配置。

## 版本

- 应用版本：1.5.1
- 镜像文件：`docker-group-manager-1.5.1.tar`
