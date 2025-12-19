

## Docker迁移

### 需求描述:
  技术栈：web nginx docker
  我是一名前端开发，略懂docker nginx等技术栈，当前前端页面已经部署在`10.6.9.73`  环境上, 
  自己为前端项目单独创建了docker容器（以nginx为镜像基础），通过docker-compose.yml的卷配置了相关nginx代理等，以及nginx相关的转发日志映射服务地址等。

### 业务要求：
  我希望复制一份一模一样的上述容器等相关配置到`10.6.137.1` 服务器上，当前服务器没有外网，不能拉去网络上的镜像，我改如何操作？

### 实现：
#### 方法一：离线镜像 + 配置迁移（推荐）

  - 源服务器 `10.6.9.73` 上识别并打包镜像
    - 进入项目目录：`cd /path/to/your/project`（即 `docker-compose.yml` 所在目录）
    - 列出 Compose 使用的镜像：`docker compose images`（如用旧版命令则 `docker-compose images`）
    - 提取镜像列表：`docker compose config | awk '/image:/ {print $2}' | sort -u > images.txt`
    - 打包镜像为离线文件：`docker save -o frontend_images.tar $(cat images.txt)`
      - 示例一：只有一个镜像
        - `images.txt` 内容：
          ```
          my-frontend:1.0
          ```
        - 命令：
          ```
          docker save -o frontend_images.tar my-frontend:1.0
          ```
      - 示例二：多个镜像
        - `images.txt` 内容：
          ```
          nginx:1.25
          node:18-alpine
          registry.example.com/web/my-frontend:1.0
          ```
        - 命令：
          ```
          docker save -o frontend_images.tar $(awk 'NF' images.txt)
          ```
        - 或：
          ```
          xargs -a images.txt -r docker save -o frontend_images.tar
          ```
      - 示例三：镜像未打标签
        - 先打标签：
          ```
          docker tag <IMAGE_ID> my-frontend:1.0
          ```
        - `images.txt` 内容：
          ```
          my-frontend:1.0
          ```
        - 命令：
          ```
          docker save -o frontend_images.tar my-frontend:1.0
          ```
      - 如服务包含 `build:` 指令，`docker compose images` 会包含构建出的镜像；确保容器已构建/运行后再执行上一步。
  - 打包项目运行所需配置与静态资源
    - 将 Compose 与环境文件打包：`tar -czf project_configs.tgz docker-compose.yml .env`
    - 同时打包 Nginx 配置与站点资源（按你的映射实际路径调整，缺失则去掉）：  
      `tar -czf nginx_and_site.tgz nginx.conf conf.d/ site/ dist/ logs/`
    - 如果卷映射到了宿主机目录（如 `./logs`、`./conf.d`），这些目录都需要一起拷贝。
  - 传输到目标服务器 `10.6.137.1`
    - 内网可达时使用 `scp`：  
      `scp frontend_images.tar project_configs.tgz nginx_and_site.tgz user@10.6.137.1:/opt/frontend`
    - 若两机不可直接连通或仅 USB/U 盘可用，切分大文件后拷贝：  
      `split -b 1024m frontend_images.tar frontend_images.tar.part.`  
      在目标机合并：`cat frontend_images.tar.part.* > frontend_images.tar`
  - 在目标服务器加载镜像并启动
    - 准备目录：`sudo mkdir -p /opt/frontend && cd /opt/frontend`
    - 加载镜像：`docker load -i frontend_images.tar`
    - 解压配置与资源：  
      `tar -xzf project_configs.tgz`  
      `tar -xzf nginx_and_site.tgz`
    - 根据 `docker-compose.yml` 中的 `volumes` 创建缺失的宿主机目录，并放置对应文件：  
      例如 `mkdir -p ./conf.d ./logs ./site`，确保与 compose 中映射的相对/绝对路径一致
    - 离线启动（避免重新构建/拉取）：  
      `docker compose up -d --no-build`（或 `docker-compose up -d`）
    - 验证：  
      `docker ps` 查看容器状态  
      `curl -I http://127.0.0.1:<映射端口>/` 返回 200/301/302 即可
  - 避免离线环境被动拉取镜像
    - 确保目标机已 `docker load` 了 compose 中引用的确切镜像标签（例如 `nginx:1.25`）。  
    - 如需强制禁止拉取，可在 compose v2 中为服务添加：  
      ```
      pull_policy: never
      ```
      例如：
      ```
      services:
        web:
          image: nginx:1.25
          pull_policy: never
      ```
  - 上游转发与文件权限
    - 如果 Nginx 反向代理了内网服务，确认这些上游地址在新环境可达，否则需更新 `conf.d/*.conf` 中的 `proxy_pass`。
    - 确保宿主机上的映射目录拥有容器可写权限（如日志目录）：`sudo chown -R 101:101 ./logs`（按镜像内用户调整；Nginx 默认通常使用 `nginx` 或 `root`）

#### 方法二：导出容器快照（获取运行时改动）

  - 适用场景：容器内有运行时改动（不在卷中）、或手工进入容器做过配置且未回写到宿主机
  - 在源服务器上将当前容器提交为镜像并保存
    - 找到容器：`docker ps | grep nginx`
    - 提交为新镜像：`docker commit <container_id> frontend-nginx:offline-snapshot`
    - 保存：`docker save -o frontend_nginx_snapshot.tar frontend-nginx:offline-snapshot`
  - 在目标服务器加载并替换 compose 中的镜像引用
    - 加载：`docker load -i frontend_nginx_snapshot.tar`
    - 将 `docker-compose.yml` 中对应服务的 `image:` 改为 `frontend-nginx:offline-snapshot`，然后：  
      `docker compose up -d --no-build`
  - 注意：`docker export` 只导出容器文件系统，丢失元数据与层；一般不推荐用来替代 `docker save`

#### 常用排错

  - 启动后无法访问
    - `docker logs <container_name> --tail 100` 检查 Nginx 报错
    - 核对端口映射 `ports:` 与目标机防火墙/安全组
  - 启动时仍尝试拉取镜像
    - 检查镜像标签是否一致；已加载的镜像需与 compose 中 `image:` 完全匹配
    - 使用 `pull_policy: never` 强制本地使用
  - 卷目录为空或缺文件
    - 重新核对源机的卷映射目录，将其完整拷贝到目标机相同路径


#### 自己总结
```sh
1. 打包镜像为离线文件：`docker save -o frontend_images.tar my-frontend:1.0`
2. docker load -i frontend_images.tar
```
