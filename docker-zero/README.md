# Docker 入门示例

## 文件说明
- `app.py`     — 最简单的 Python HTTP 服务，返回 JSON
- `Dockerfile` — 打包配置

## 第一次运行（完整流程）

```bash
# 1. 构建镜像
docker build -t my-app .

# 2. 跑起来（后台运行，8080 端口）
docker run -d -p 8080:8080 --name my-container my-app

# 3. 验证
curl http://localhost:8080
```

## 常用命令速查

```bash
docker ps                        # 看运行中的容器
docker logs my-container         # 看日志
docker logs -f my-container      # 实时追踪日志
docker stop my-container         # 停止
docker start my-container        # 重新启动
docker rm my-container           # 删除容器（需先 stop）
docker images                    # 查看本地所有镜像
docker rmi my-app                # 删除镜像
```

## 扩容示例（手动起多个实例）

```bash
docker run -d -p 8081:8080 --name app-1 my-app
docker run -d -p 8082:8080 --name app-2 my-app
docker run -d -p 8083:8080 --name app-3 my-app

# 验证三个都在跑
docker ps
```

## 清理所有容器

```bash
docker stop app-1 app-2 app-3
docker rm app-1 app-2 app-3
```
