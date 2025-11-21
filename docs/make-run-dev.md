## make run-dev 说明

**make run-dev** 实际执行：

```bash
docker compose -f deploy/docker-compose/docker-compose-dev.yaml up
```

所有服务都由该 Compose 编排栈统一管理。

## 常用运维命令

- **查看当前状态**

  ```bash
  cd /Users/nengli/projects/js_projects/miniprogram/bee-api/deploy/docker-compose
  docker compose -f docker-compose-dev.yaml ps
  ```

- **查看实时日志**

  ```bash
  docker compose -f docker-compose-dev.yaml logs -f server  # 其他服务替换为对应名称
  ```

- **后台运行（将前台栈切到后台）**

  ```bash
  docker compose -f docker-compose-dev.yaml up -d
  ```

- **重启单个服务**

  ```bash
  docker compose -f docker-compose-dev.yaml restart server  # 或 web / mysql / redis
  ```

- **重启全部服务**

  ```bash
  docker compose -f docker-compose-dev.yaml restart
  ```

- **关闭并移除容器**

  ```bash
  # 若要同时删除数据卷（会清空 MySQL/Redis 数据），加上 -v 参数
  docker compose -f docker-compose-dev.yaml down
  docker compose -f docker-compose-dev.yaml down -v
  ```

- **更新镜像或代码**

  - **更新代码并重新构建运行：**

    ```bash
    # 同步代码后执行
    docker compose -f docker-compose-dev.yaml up -d --build
    ```

  - **仅更新依赖镜像：**

    ```bash
    docker compose -f docker-compose-dev.yaml pull
    docker compose -f docker-compose-dev.yaml up -d
    ```

上述命令在服务器上执行，即可维护、重启或关闭通过 **make run-dev** 启动的容器。