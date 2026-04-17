# Docker & Kubernetes 运维操作手册

> 面向运维新手的快速上手指南，从 Docker 单机到 K8s 集群管理。
> 读完这份文档，你能独立完成：打包镜像、部署服务、扩缩容、排查问题。

---

## 目录

1. [核心概念（最少必要知识）](#一核心概念最少必要知识)
2. [Docker 快速上手](#二docker-快速上手)
3. [Docker 常用命令速查](#三docker-常用命令速查)
4. [单机扩容与端口映射](#四单机扩容与端口映射)
5. [镜像跨机器迁移](#五镜像跨机器迁移)
6. [Kubernetes 快速上手](#六kubernetes-快速上手)
7. [K8s 常用命令速查](#七k8s-常用命令速查)
8. [K8s 扩缩容](#八k8s-扩缩容)
9. [常见问题排查](#九常见问题排查)
10. [进阶路线图](#十进阶路线图)

---

## 一、核心概念（最少必要知识）

### 1.1 一个类比帮你记住所有概念

| 概念 | 类比 | 作用 |
|------|------|------|
| 镜像 Image | 速冻包（静态、只读） | 应用 + 环境打包好的模板 |
| 容器 Container | 加热后的菜（活的实例） | 镜像跑起来的运行实例 |
| Docker | 微波炉 | 运行容器的工具 |
| K8s | 中央厨房调度系统 | 管理一大堆容器的平台 |

### 1.2 关键区别

- **一个镜像可以起多个容器**（就像一个速冻包可以热多份）
- **容器不是小虚拟机**：虚拟机带完整操作系统，很重；容器共享宿主机内核，很轻
- **Dockerfile 是造镜像的菜谱**，告诉 Docker 如何一步步构建镜像

### 1.3 什么时候用什么

```
单个小项目           →  Docker
多服务本地开发       →  Docker Compose
生产环境高可用       →  Kubernetes（K8s）
```

---

## 二、Docker 快速上手

### 2.1 最简测试服务（Python）

`app.py`：

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import json
from datetime import datetime

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        body = json.dumps({
            "message": "Hello, Docker!",
            "path": self.path,
            "time": datetime.now().isoformat()
        }).encode()
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.end_headers()
        self.wfile.write(body)

HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
```

### 2.2 最简 Dockerfile

```dockerfile
# 基础镜像
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 复制代码
COPY app.py .

# 启动命令
CMD ["python", "app.py"]
```

### 2.3 构建与运行

```bash
# 1. 构建镜像（最后的 . 表示用当前目录的 Dockerfile）
docker build -t my-app .

# 2. 运行容器
docker run -d -p 8080:8080 --name my-container my-app

# 3. 验证
curl http://localhost:8080
```

参数说明：
- `-d`：后台运行（detached）
- `-p 8080:8080`：`宿主机端口:容器内端口`，外面访问前一个，流量转到后一个
- `--name`：给容器起名字，不然会生成随机名（如 `quirky_einstein`）

---

## 三、Docker 常用命令速查

### 3.1 查看与监控

```bash
docker ps                        # 查看运行中的容器
docker ps -a                     # 查看所有容器（含已停止）
docker images                    # 查看本地所有镜像
docker logs <容器名>              # 查看容器日志
docker logs -f <容器名>           # 实时追踪日志
docker stats                     # 查看容器资源使用（CPU/内存）
```

### 3.2 生命周期管理

```bash
docker start <容器名>             # 启动已停止的容器
docker stop <容器名>              # 停止容器
docker restart <容器名>           # 重启
docker rm <容器名>                # 删除容器（需先 stop）
docker rmi <镜像名>               # 删除镜像
```

### 3.3 进入容器排查

```bash
docker exec -it <容器名> sh       # 进入容器的 shell
docker exec -it <容器名> bash     # 如果容器有 bash
```

### 3.4 清理（小心使用）

```bash
docker container prune            # 清理所有已停止的容器
docker image prune                # 清理悬空镜像
docker system prune -a            # 清理所有未使用资源（谨慎！）
```

---

## 四、单机扩容与端口映射

### 4.1 手动扩容（起多个容器）

```bash
# 起 3 个容器，映射到不同宿主机端口
docker run -d -p 8081:8080 --name app-1 my-app
docker run -d -p 8082:8080 --name app-2 my-app
docker run -d -p 8083:8080 --name app-3 my-app

# 验证
docker ps
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
```

### 4.2 端口映射规则（新手常踩坑）

```
格式：-p 宿主机端口:容器内端口

-p 8081:8080  →  访问宿主机 8081，流量转到容器内 8080
```

**常见错误**：容器映射到 8081，却 curl 8080，提示连不上。

### 4.3 命名规则

- `--name` 必须**唯一**，重名会报错 `Conflict. The container name is already in use`
- 单机扩容时推荐格式：`app-1`、`app-2` 或 `my-app-prod`
- 到了 Docker Compose / K8s 后，不需要手动起名

---

## 五、镜像跨机器迁移

场景：Mac 本地访问 Docker Hub 失败（国内网络问题），需要从服务器拷贝镜像到本地。

### 5.1 导出 / 导入镜像

```bash
# 在有镜像的机器上执行（打包并压缩）
docker save my-app | gzip > my-app.tar.gz

# 传到目标机器
scp root@server-ip:/path/my-app.tar.gz ~/Desktop/

# 在目标机器上加载
docker load < ~/Desktop/my-app.tar.gz

# 验证
docker images
```

### 5.2 配置国内镜像源（根治方案）

Docker Desktop：Settings → Docker Engine，加上：

```json
{
  "registry-mirrors": [
    "https://dockerpull.org",
    "https://docker.rainbond.cc"
  ]
}
```

点 Apply & Restart 即可。

---

## 六、Kubernetes 快速上手

### 6.1 为什么需要 K8s

Docker 单机的局限：容器挂了要人工拉起，扩缩容要手动，多机器管理混乱。

K8s 解决的事：
- 容器挂了自动重启
- 一条命令扩缩容
- 滚动更新、灰度发布
- 自动负载均衡
- 资源调度优化

### 6.2 服务器配置要求

| 项目 | 最低要求 | 说明 |
|------|---------|------|
| CPU | 2 核 | - |
| 内存 | 2 GB | 仅控制平面本身 |
| Swap | 必须关闭 | K8s 要求 |
| OS | Linux 64 位 | 推荐 Ubuntu 22/24 |

**提示**：生产环境建议 4 核 8 GB 起步，和业务服务分开部署。

### 6.3 本地学习环境

**推荐**：Mac 上用 Docker Desktop 自带的 K8s
- 打开 Docker Desktop → Settings → Kubernetes → Enable Kubernetes
- 等左下角显示绿色，执行 `kubectl get nodes` 看到 Ready 即可

**其他选项**：minikube、kind、k3s（资源受限场景）

### 6.4 核心概念

| 概念 | 作用 |
|------|------|
| Pod | K8s 最小调度单位，通常里面 1 个容器 |
| Deployment | 声明"我要几个副本"，自动维持 |
| ReplicaSet | Deployment 内部的副本管理器（一般不手动操作） |
| Service | 给 Pod 一个稳定的访问入口（负载均衡） |
| Ingress | 把外部域名路由到 Service |
| ConfigMap | 普通配置（环境变量等） |
| Secret | 敏感配置（密码、Token） |

### 6.5 部署 my-app 的完整示例

`deployment.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: my-app:latest
        imagePullPolicy: Never      # 本地镜像，不去网上拉
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

`service.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
  type: NodePort
```

部署流程：

```bash
# 应用配置
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 查看状态
kubectl get pods
kubectl get deployments
kubectl get services

# 访问（NodePort 会分配一个 3xxxx 端口）
kubectl get services
curl http://localhost:<NodePort>
```

### 6.6 K8s 工作机制的核心理解

K8s 是**声明式**的：你说"我要 3 个副本"，K8s 永远盯着这个目标。

```
Docker：   你 → 手动起容器 → 挂了就没了
K8s：      你 → 声明期望状态 → K8s 永远维持
```

Service 通过**标签**找 Pod（`app=my-app`），不依赖 Pod 名字，因此 Pod 重建后 IP 变化不影响访问。

---

## 七、K8s 常用命令速查

### 7.1 查看状态（最常用）

```bash
kubectl get nodes                 # 看集群节点
kubectl get pods                  # 看 Pod
kubectl get pods -A               # 看所有命名空间
kubectl get deployments           # 看 Deployment
kubectl get services              # 看 Service
kubectl get all                   # 一次看全（Pod/Deploy/Svc/RS）
kubectl describe pod <pod名>       # 看详情，Events 区域最关键
```

### 7.2 部署与更新

```bash
kubectl apply -f deployment.yaml                          # 应用单个 yaml
kubectl apply -f .                                        # 应用当前目录所有 yaml
kubectl rollout status deployment/my-app                  # 查看发布进度
kubectl rollout undo deployment/my-app                    # 回滚到上一版本
kubectl set image deployment/my-app my-app=my-app:v2      # 更新镜像版本
```

### 7.3 调试排查

```bash
kubectl logs <pod名>              # 查看日志
kubectl logs -f <pod名>           # 实时追踪
kubectl logs <pod名> --previous   # 上次崩溃的日志
kubectl exec -it <pod名> -- sh    # 进入容器
kubectl get events --sort-by=.lastTimestamp   # 按时间看事件
kubectl port-forward pod/<pod名> 8080:8080    # 端口转发到本地
```

### 7.4 配置管理

```bash
kubectl create configmap my-config --from-literal=ENV=prod
kubectl create secret generic my-secret --from-literal=DB_PASSWORD=xxx
kubectl get configmap
kubectl get secret
kubectl get namespace
```

### 7.5 删除资源（谨慎）

```bash
kubectl delete pod <pod名>              # Deployment 管的 Pod 会自动重建
kubectl delete deployment my-app        # 删除 Deployment 及所有 Pod
kubectl delete -f deployment.yaml       # 删除 yaml 定义的资源
```

---

## 八、K8s 扩缩容

### 8.1 手动扩缩容

```bash
kubectl scale deployment my-app --replicas=5    # 扩到 5 个
kubectl scale deployment my-app --replicas=1    # 缩到 1 个
```

### 8.2 自动扩缩容（HPA）

```bash
# CPU 超过 50% 自动扩，最少 2 个，最多 10 个
kubectl autoscale deployment my-app --min=2 --max=10 --cpu-percent=50

# 查看 HPA 状态
kubectl get hpa
```

K8s 每 15 秒检查一次 CPU 使用率，自动调整副本数。

---

## 九、常见问题排查

### 9.1 Docker 镜像拉取超时

**现象**：`failed to fetch ... i/o timeout`

**原因**：国内无法访问 Docker Hub

**解决**：
1. 配置国内镜像源（见第 5.2 节）
2. 从其他机器用 `docker save/load` 导入

### 9.2 端口无法访问

**排查步骤**：

```bash
# 1. 确认容器在跑
docker ps

# 2. 确认端口映射正确（用映射后的宿主机端口访问）
docker port <容器名>

# 3. 确认端口未被占用
lsof -i:8080

# 4. 查看日志
docker logs <容器名>
```

### 9.3 K8s Pod 一直 Pending / CrashLoopBackOff

**必做三步**：

```bash
kubectl describe pod <pod名>        # 看 Events 报什么错
kubectl logs <pod名>                # 看应用日志
kubectl logs <pod名> --previous     # 看崩溃前的日志
```

常见原因：
- 镜像拉不到 → 检查 `imagePullPolicy` 和镜像名
- 资源不够 → 节点内存/CPU 不足
- 应用本身报错 → 看 logs

### 9.4 容器名冲突

**现象**：`Conflict. The container name "/app-1" is already in use`

**解决**：

```bash
docker rm <已存在容器名>     # 或换个名字
```

### 9.5 服务器资源不足

```bash
top                         # 看 CPU/内存使用
df -h                       # 看磁盘
docker stats                # 看每个容器的资源占用
```

如果 Docker 容器占用过多，可以设置资源限制：

```bash
docker run -d --memory=128m --cpus=0.5 my-app
```

---

## 十、进阶路线图

按这个顺序学，每个阶段都能直接用于工作：

```
阶段 1（当前）   熟练 Docker 单机操作
             ↓    ·优化 Dockerfile（多阶段构建、缓存）
                  ·学会压测和资源限制

阶段 2         学 Docker Compose
             ↓    ·用一个 yaml 管多服务（Next.js + Python + Redis）
                  ·这是 K8s 的预热

阶段 3         K8s 基础
             ↓    ·本地 Docker Desktop 实践
                  ·Deployment / Service / ConfigMap / Secret

阶段 4         监控与可观测性
             ↓    ·Prometheus + Grafana（监控告警）
                  ·Loki 或 ELK（日志收集）

阶段 5         发布流程与高可用
                  ·滚动更新、灰度发布、自动回滚
                  ·健康检查（liveness / readiness probe）
```

### 保证系统稳定性的 5 件事

1. **监控告警**：CPU、内存、响应时间、错误率都要有图有报警
2. **日志规范**：能快速定位到哪个容器哪一行代码出问题
3. **健康检查**：liveness / readiness probe 配对，挂了自动重启
4. **资源限制**：给每个容器设 CPU/内存上限，防止互相影响
5. **发布流程**：灰度发布 + 回滚机制，不要一次性全量更新

---

## 附录 A：一页命令速查（打印版）

### Docker

```bash
# 构建与运行
docker build -t <名字> .
docker run -d -p <外>:<内> --name <名字> <镜像>

# 查看
docker ps                 docker logs -f <容器>
docker images             docker stats

# 操作
docker stop <容器>         docker restart <容器>
docker rm <容器>           docker rmi <镜像>
docker exec -it <容器> sh

# 迁移
docker save <镜像> | gzip > x.tar.gz
docker load < x.tar.gz
```

### Kubernetes

```bash
# 部署
kubectl apply -f <file>.yaml
kubectl delete -f <file>.yaml

# 查看
kubectl get pods                      kubectl get svc
kubectl get deployments               kubectl get all
kubectl describe pod <名>

# 调试
kubectl logs -f <pod>
kubectl exec -it <pod> -- sh
kubectl port-forward pod/<名> 8080:8080

# 扩缩容
kubectl scale deployment <名> --replicas=N
kubectl autoscale deployment <名> --min=2 --max=10 --cpu-percent=50

# 更新与回滚
kubectl set image deployment/<名> <容器名>=<新镜像>
kubectl rollout status deployment/<名>
kubectl rollout undo deployment/<名>
```

---

## 附录 B：参考资源

- Docker 官方文档：https://docs.docker.com
- Kubernetes 官方文档：https://kubernetes.io/docs
- Docker 国内镜像源：https://dockerpull.org
- k3s（轻量级 K8s）：https://k3s.io

---

*本手册持续更新，欢迎补充实战经验。*
