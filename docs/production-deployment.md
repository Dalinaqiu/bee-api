# Bee API 生产部署指南

本文档说明如何在一台 Linux 服务器上以 Docker Compose 方式部署 bee-api（含管理后台与依赖服务）。完成后，可通过 nginx 或自带端口对外暴露服务。

## 1. 环境与前提
- Linux 服务器（推荐 4C+8G，预留 30G 磁盘用于镜像与数据卷）
- 已安装 `git`、`Docker Engine 20+`、`docker compose v2`
- 可以访问容器镜像仓库（默认使用官方镜像，若服务器无法访问公网需配置代理）
- 已准备域名与 HTTPS 证书（可选，用于反向代理）

> 若服务器上已有 MySQL/Redis，需要先调整 `deploy/docker-compose/docker-compose.yaml`，避免端口冲突或重复部署依赖。

## 2. 获取代码
```bash
git clone https://github.com/woniudiancang/bee-api.git
cd bee-api
```
后续命令均在仓库根目录执行。

## 3. 准备配置与数据
1. **后端配置**：编辑 `server/config.docker.yaml`（或复制为独立文件）并根据实际情况修改：
   - `mysql`、`redis` 连接保持为 `gva-mysql:3306` 和 `gva-redis:6379`（容器内互通主机名）
   - `bee-shop.host`：填入生产域名，用于回调与小程序请求
   - 其他支付、OSS、第三方服务密钥
2. **初始化 SQL**：将需要的初始 SQL（如 `bee-region.sql`、业务数据）准备好。部署后可通过 MySQL 容器导入，命令示例：
   ```bash
   docker exec -i gva-mysql mysql -ugva -pAa@6447985 qmPlus < /path/to/bee-region.sql
   ```
3. **自定义环境变量**：在 `deploy/docker-compose/docker-compose.yaml` 中：
   - `mysql` 服务建议取消注释 `MYSQL_ROOT_PASSWORD` 并改为强密码
   - 如需对外暴露不同端口，调整 `ports` 映射，例如 `- "80:8080"` 暴露前端

## 4. 构建与启动
`deploy/docker-compose/docker-compose.yaml` 会为 web、server、mysql、redis 构建/下载镜像、挂载数据卷（`mysql:/var/lib/mysql`、`redis:/data`），并放在自定义网络 `177.7.0.0/16`。执行：
```bash
cd deploy/docker-compose
docker compose up -d --build
```
命令结束后拥有以下服务：
- `gva-web`：基于 `web/Dockerfile` 生成，默认监听 8080
- `gva-server`：使用 `server/Dockerfile` 编译出的 Go 可执行文件，监听 8888
- `gva-mysql`、`gva-redis`：分别持久化到 Docker 卷，可在 `/var/lib/docker/volumes/<project>_mysql/_data` 等路径找到

## 5. 健康检查与初始化
1. 查看容器状态：
   ```bash
   docker compose ps
   docker logs -f gva-server
   ```
2. 打开浏览器访问 `http://<服务器IP>:8888`，按照引导完成数据库、管理员账号初始化。
3. 将管理后台地址配置给小程序端，并在后台中配置微信相关参数。

## 6. 运维操作
- **重启单个服务**：`docker compose restart server`
- **更新代码**：拉取最新代码后重新执行 `docker compose up -d --build`
- **备份数据**：
  - MySQL：`docker exec gva-mysql mysqldump -ugva -pAa@6447985 qmPlus > backup.sql`
  - Redis：数据已持久化到卷，可通过 `docker run --rm -v <卷>:/data alpine tar czf /backup/redis-data.tgz /data` 备份
- **日志**：`docker logs -f gva-web`、`docker logs -f gva-server`
- **扩展**：如需将 web/server 划分至独立主机，可拆分 Compose，单独部署数据库集群或使用云数据库。

## 7. 对外暴露与安全
1. 建议使用 nginx/Traefik 做反向代理，将域名指向服务器并转发到 `gva-web`、`gva-server`。
2. 修改默认数据库密码、Redis 密码（可在 `deploy/docker-compose/docker-compose.yaml` 添加 `requirepass`），并在安全组/防火墙中仅开放需对外的端口。
3. 定期升级基础镜像（`mysql`, `redis`, `golang`, `node`）并重新构建。

## 8. 常见问题
- **端口被占用**：检查 `8080/8888/13306/16379` 是否已有服务使用，如有冲突调整 Compose 中的映射。
- **镜像下载缓慢**：在服务器上配置国内镜像加速（如 `registry.cn-hangzhou.aliyuncs.com`）。
- **容器内访问宿主机服务**：使用 `host.docker.internal` 或在 Compose 中挂载需要的网络。

完成以上步骤后，bee-api 的生产环境即可投入使用，后续只需在仓库根目录执行 `docker compose up -d --build` 即可完成更新。
