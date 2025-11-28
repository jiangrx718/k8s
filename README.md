# k8s 本地配置

本仓库用于在本地开发环境部署数据库与中间件（`mysql`、`redis`、`rabbitmq`），以及应用服务 `document-service`。采用 `kustomize` 组织资源，按功能划分为 `db/` 与 `app/` 两个目录。

## 项目结构
- `db/`：数据库与中间件（`mysql`、`redis`、`rabbitmq`）的 `Deployment`、`Service`、`PVC` 等资源
- `app/`：应用服务 `document-service` 的 `ConfigMap`、`Secret`、`Deployment`、`Service`
- `*-nodeport.yaml`：用于在本地通过 `NodePort` 暴露端口，便于直接从宿主机访问
- `kustomization.yaml`：使用 `kubectl apply -k` 按目录管理与部署

## 命名空间
- 数据库与中间件使用命名空间 `db`
- 应用服务使用命名空间 `app`

首次部署前创建命名空间：

```bash
kubectl create namespace db
kubectl create namespace app
```

## 数据库与中间件配置（db/）
- `mysql`：`Deployment` + `Service(ClusterIP)`，数据卷通过 `PVC` 持久化
  - Service 名称：`mysql-service.db.svc.cluster.local`，端口 `3306`
- `redis`：`Deployment` + `ConfigMap` 加载 `redis.conf`，`PVC` 持久化
  - Service 名称：`redis-service.db.svc.cluster.local`，端口 `6379`
  - 配置包含密码与持久化，实际使用时请修改默认密码
- `rabbitmq`：`Deployment` + `Service(LoadBalancer)` 默认对外暴露管理端与 AMQP
  - Service 名称：`rabbitmq-service.db.svc.cluster.local`，端口 `5672`（AMQP）与 `15672`（管理）
  - 本地环境可改用 `rabbitmq-service-nodeport.yaml` 通过 `NodePort` 暴露端口

## 应用服务配置（app/）
- `document-service`：`Deployment` + `Service(NodePort)`，容器端口与 Service 端口均为 `8088`
- 依赖的连接信息通过环境变量注入：
  - `SPRING_DATASOURCE_URL` 指向 `mysql-service.db.svc.cluster.local:3306`
  - `SPRING_REDIS_HOST` 指向 `redis-service.db.svc.cluster.local:6379`
  - `SPRING_RABBITMQ_HOST` 指向 `rabbitmq-service.db.svc.cluster.local:5672`
- `Secret` 与 `ConfigMap` 提供必要的凭据与配置，实际使用时请根据环境更新值

## 本地访问端口（NodePort）
- `document-service`：`30088`（映射容器 `8088`）
- `mysql`：`30036`
- `redis`：`30379`
- `rabbitmq`：`30672`（AMQP）、`31672`（管理）

如果仅使用集群内访问（`ClusterIP`），请在宿主机通过 `kubectl port-forward` 或 Ingress/LoadBalancer 方式访问。

## 部署与清理

部署（使用 kustomize）：

```bash
kubectl apply -k db/
kubectl apply -k app/
```

切换为本地 NodePort 暴露（示例以 rabbitmq 为例）：

```bash
kubectl apply -f db/rabbitmq-service-nodeport.yaml
```

查看资源：

```bash
kubectl get all -n db
kubectl get all -n app
```

清理：

```bash
kubectl delete -k app/
kubectl delete -k db/
```

## 常用 kubectl 命令

- 基础操作
  - `kubectl apply -k <dir>`：按目录应用资源（kustomize）
  - `kubectl apply -f <file>`：应用单个或多个 YAML 文件
  - `kubectl delete -k <dir>` / `kubectl delete -f <file>`：删除资源

- 资源查看
  - `kubectl get pods -n <ns>` / `kubectl get svc -n <ns>`：查看 Pod 与 Service
  - `kubectl describe pod <pod> -n <ns>`：查看 Pod 详细信息与事件
  - `kubectl get pvc,pv -n <ns>`：查看存储资源

- 调试与日志
  - `kubectl logs <pod> -n <ns>`：查看日志
  - `kubectl logs -f <pod> -c <container> -n <ns>`：跟随指定容器日志
  - `kubectl exec -it <pod> -n <ns> -- /bin/sh`：进入容器交互调试
  - `kubectl port-forward svc/<svc> <local_port>:<target_port> -n <ns>`：端口转发到本机

- 部署管理
  - `kubectl rollout restart deployment/<name> -n <ns>`：重启 Deployment
  - `kubectl scale deployment/<name> --replicas=<n> -n <ns>`：水平扩缩容
  - `kubectl get events -n <ns>`：查看事件，定位异常

- 命名空间与上下文
  - `kubectl create namespace <ns>` / `kubectl delete namespace <ns>`
  - `kubectl config get-contexts` / `kubectl config use-context <ctx>`
  - `kubectl config view`：查看当前 kubeconfig

- YAML 与解释
  - `kubectl explain <resource>`：查看资源字段说明，如 `kubectl explain deployment.spec.template`
  - `kubectl diff -k <dir>`：比较即将应用的变更

> 提示：首次启动可能需要等待镜像拉取与容器就绪，若健康检查开启请适当调整 `initialDelaySeconds` 与资源请求/限制以匹配本地环境性能。

