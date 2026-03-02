# 阿里云服务器 Docker 运行说明

本目录通过 Docker Compose 管理以下服务：
- MySQL 8（端口 12306→3306，数据卷 mysql_data）
- Redis 7（端口 16379→6379，数据卷 redis_data）
- Nginx（端口 80/443，挂载本地配置与静态目录）
- MinIO（端口 9000/9001，数据卷 minio_data）

**外部网络**：所有服务加入 external 网络 common-app-net（需提前创建）。

## 目录结构
- docker-compose.yml
- nginx/
  - nginx.conf
  - conf.d/ 下站点配置：m.conf、wechat.conf、oss.conf
- redis/redis.conf

## 前置条件
1) 已安装 Docker 与 Compose（支持 docker compose v2）
2) 创建外部网络

```bash
docker network create common-app-net
```

3) 准备环境变量文件 .env（与 docker-compose.yml 同目录）

```bash
MYSQL_ROOT_PASSWORD=请设置安全密码
REDIS_PASSWORD=请设置安全密码
TZ=Asia/Shanghai
```

## 启动与关闭
- 启动（首次或更新后）

```bash
docker compose up -d
```

- 查看状态

```bash
docker compose ps
```

- 停止

```bash
docker compose stop
```

- 关闭并移除容器（保留数据卷）

```bash
docker compose down
```

- 关闭并移除容器与数据卷（谨慎）

```bash
docker compose down -v
```

## 重启与热加载
- 重启全部服务

```bash
docker compose restart
```

- 重启指定服务（示例：仅重启 nginx）

```bash
docker compose restart nginx
```

- 热加载 Nginx 配置（无需重启容器）

```bash
docker exec -it nginx nginx -s reload
```

## 日志查看
- 全部服务实时日志

```bash
docker compose logs -f
```

- 指定服务实时日志（示例：nginx、mysql、redis、minio）

```bash
docker compose logs -f nginx
docker compose logs -f mysql
docker compose logs -f redis
docker compose logs -f minio
```

## 常用信息
- MySQL
  - 地址：127.0.0.1:12306
  - 帐号：root，密码来自 .env 中的 MYSQL_ROOT_PASSWORD
- Redis
  - 地址：127.0.0.1:16379
  - 密码：来自 .env 中的 REDIS_PASSWORD
- MinIO
  - API：:9000，Console：:9001
  - 默认账号密码：admin / 12345678
- Nginx
  - 主配置：nginx/nginx.conf
  - 站点：nginx/conf.d/*.conf（m.conf、wechat.conf、oss.conf）
  - 静态目录挂载：服务器 /data/static → 容器 /usr/share/nginx/html/static

## 升级镜像

```bash
docker compose pull
docker compose up -d
```

## 注意事项
- 确保外部网络 common-app-net 已创建，否则容器无法加入网络
- 若占用 80/443/12306/16379/9000/9001，请在 docker-compose.yml 中调整映射端口
- 修改配置文件后建议先校验格式，再 reload 或重启对应服务
