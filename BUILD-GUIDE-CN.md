# 从源码编译到 Kubernetes 部署完整指南

[English](./BUILD-GUIDE.md) | 简体中文

> **文档状态**: 🚧 正在编写中...

## 概述

本指南面向需要从源码编译 AI Gateway 核心组件的技术人员，涵盖从源码获取、编译构建、Docker 镜像制作，到推送镜像仓库并集成到 Kubernetes 部署的完整工作流程。

### 涵盖的组件

本指南详细说明以下三个核心组件的编译部署流程：

1. **BFE (数据面)** - 负责流量转发与接入控制
   - GitHub: https://github.com/bfenetworks/bfe
   
2. **AI Gateway API (控制面)** - 负责策略/配置下发接口
   - GitHub: https://github.com/yf-networks/ai-gateway-api
   
3. **Service Controller** - 负责发现并同步后端服务
   - GitHub: https://github.com/bfenetworks/service-controller

---

## 目录

- [前置条件](#前置条件)
- [BFE 编译](#bfe-编译)
- [AI Gateway API 编译](#ai-gateway-api-编译)
- [Service Controller 编译](#service-controller-编译)
- [Kubernetes 部署集成](#kubernetes-部署集成)
- [完整示例](#完整示例)
- [故障排查](#故障排查)
- [参考资料](#参考资料)

---

## 前置条件

### 开发环境要求

在开始编译之前，请确保您的系统满足以下要求：

#### Go 环境

- **版本要求**: Go 1.21 或更高版本
- **安装验证**:
  ```bash
  go version
  # 应输出: go version go1.21.x ...
  ```
- **安装指南**: 访问 [Go 官方网站](https://go.dev/dl/) 下载安装

#### Docker 环境

- **版本要求**: 支持 Docker Buildx 的版本（Docker Desktop 20.10+ 或 Docker Engine 19.03+）
- **验证 Docker**:
  ```bash
  docker --version
  # 应输出: Docker version 20.10.x ...
  ```
- **验证 Buildx**:
  ```bash
  docker buildx version
  # 应输出: github.com/docker/buildx vX.X.X ...
  ```
- **安装指南**: 
  - macOS/Windows: 安装 [Docker Desktop](https://www.docker.com/products/docker-desktop)
  - Linux: 参考 [Docker Engine 安装文档](https://docs.docker.com/engine/install/)

#### Git 环境

- **版本要求**: Git 2.0 或更高版本
- **验证安装**:
  ```bash
  git --version
  ```

### 可选工具

#### kubectl（用于验证 Kubernetes 部署）

```bash
kubectl version --client
```

安装指南: [Kubernetes 文档](https://kubernetes.io/docs/tasks/tools/)

#### 镜像仓库账号

如需推送镜像到远程仓库，需要准备以下之一：
- [GitHub Container Registry (ghcr.io)](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Docker Hub](https://hub.docker.com/)
- 私有镜像仓库

登录命令示例：
```bash
# GitHub Container Registry
docker login ghcr.io -u <username>

# Docker Hub
docker login -u <username>
```

### 网络环境配置

#### Go 代理配置（国内网络强烈推荐）

为了加速 Go 模块下载，建议配置 GOPROXY：

```bash
# 临时设置（当前终端有效）
export GOPROXY=https://goproxy.cn,direct

# 或使用官方代理
export GOPROXY=https://proxy.golang.org,direct

# 永久设置（添加到 ~/.bashrc 或 ~/.zshrc）
echo 'export GOPROXY=https://goproxy.cn,direct' >> ~/.bashrc
source ~/.bashrc
```

验证配置：
```bash
go env GOPROXY
# 应输出: https://goproxy.cn,direct
```

#### Docker 镜像加速（可选）

国内用户可配置 Docker 镜像加速器，编辑 `/etc/docker/daemon.json`（Linux）或通过 Docker Desktop 设置（macOS/Windows）：

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

重启 Docker 服务使配置生效。

---

## BFE 编译

BFE（Beyond Front End）是数据面核心组件，负责流量转发与接入控制。

### 1. 源码获取

**GitHub 仓库**: [https://github.com/bfenetworks/bfe](https://github.com/bfenetworks/bfe)

克隆源码：

```bash
# 克隆仓库
git clone https://github.com/bfenetworks/bfe.git
cd bfe

# 推荐：切换到稳定分支或最新 release tag
git checkout develop  # 或 git checkout v1.8.0
```

### 2. 本地镜像构建

在仓库根目录执行：

```bash
make docker
```

**说明**:
- 此命令会构建 **prod（生产）** 和 **debug（调试）** 两个版本的镜像
- 镜像 tag 来自仓库根目录的 `VERSION` 文件
- tag 会自动规范化为 `v` 前缀（例如：`1.8.0` → `v1.8.0`）

**构建产物**（以 VERSION=1.8.0 为例）:
```
bfe:v1.8.0        # 生产版本
bfe:v1.8.0-debug  # 调试版本
bfe:latest        # 始终指向最新的生产版本
```

**自定义镜像名**（可选）:

```bash
make docker BFE_IMAGE_NAME=my-custom-bfe
# 生成: my-custom-bfe:v1.8.0, my-custom-bfe:latest
```

### 3. 验证镜像

#### 查看构建的镜像

```bash
docker images | grep bfe
```

应显示类似输出：
```
bfe    v1.8.0        <image-id>   <size>   <time>
bfe    v1.8.0-debug  <image-id>   <size>   <time>
bfe    latest        <image-id>   <size>   <time>
```

#### 运行镜像验证

```bash
docker run --rm \
  -p 8080:8080 \
  -p 8443:8443 \
  -p 8421:8421 \
  bfe:latest
```

**验证服务**:
- 访问监控端点: http://127.0.0.1:8421/monitor
- 访问服务端点: http://127.0.0.1:8080/ （如配置未命中可能返回 500，属正常）

### 4. 多架构镜像构建与推送

如需将镜像推送到远程仓库供 Kubernetes 集群使用，执行：

```bash
make docker-push REGISTRY=<your-registry>
```

**必填参数**:
- `REGISTRY`: 镜像仓库前缀（如 `ghcr.io/your-org`）

**可选参数**:
- `PLATFORMS`: 构建平台（默认 `linux/amd64,linux/arm64`）
- `BFE_IMAGE_NAME`: 镜像名（默认 `bfe`）

**示例**:

```bash
# 推送到 GitHub Container Registry（多架构）
make docker-push REGISTRY=ghcr.io/cc14514

# 推送到私有仓库（仅 amd64）
make docker-push \
  REGISTRY=registry.example.com \
  PLATFORMS=linux/amd64

# 自定义镜像名
make docker-push \
  REGISTRY=ghcr.io/myorg \
  BFE_IMAGE_NAME=team/bfe
```

**推送的镜像 tag**（以 VERSION=1.8.0 为例）:
```
ghcr.io/cc14514/bfe:v1.8.0
ghcr.io/cc14514/bfe:v1.8.0-debug
ghcr.io/cc14514/bfe:latest
```

### 5. 镜像内部结构

BFE 镜像的关键目录：

```
/home/work/bfe/conf/           # BFE 配置目录
/home/work/bfe/log/            # BFE 日志目录
/home/work/conf-agent/conf/    # conf-agent 配置目录
/home/work/conf-agent/log/     # conf-agent 日志目录
```

**自定义配置（通过挂载）**:

```bash
docker run --rm \
  -p 8080:8080 -p 8443:8443 -p 8421:8421 \
  -v $(pwd)/my-conf:/home/work/bfe/conf \
  -v $(pwd)/my-logs:/home/work/bfe/log \
  bfe:latest
```

---

## AI Gateway API 编译

AI Gateway API 是控制面核心组件，负责策略和配置下发接口。

### 1. 源码获取

**GitHub 仓库**: [https://github.com/yf-networks/ai-gateway-api](https://github.com/yf-networks/ai-gateway-api)

克隆源码：

```bash
git clone https://github.com/yf-networks/ai-gateway-api.git
cd ai-gateway-api
```

### 2. 本地镜像构建

在仓库根目录执行：

```bash
make docker
```

**关键参数**: `DASHBOARD_VERSION`（可选）

指定控制面 Dashboard 前端资源的版本（来自 [yf-networks/ai-gateway-web](https://github.com/yf-networks/ai-gateway-web) 的 release）：

```bash
make docker DASHBOARD_VERSION=v0.0.1
```

**构建产物**:
```
ai-gateway-api:v<Version>  # 版本镜像
ai-gateway-api:latest      # 最新版本
```

### 3. 验证镜像

#### 查看构建的镜像

```bash
docker images | grep ai-gateway-api
```

#### 运行镜像验证

```bash
docker run -d \
  --name ai-gateway-api \
  -p 8183:8183 \
  ai-gateway-api:latest
```

**验证服务**:
- 访问 API 端点: http://localhost:8183
- 检查日志: `docker logs ai-gateway-api`

**停止容器**:
```bash
docker stop ai-gateway-api
docker rm ai-gateway-api
```

### 4. 镜像推送

推送到远程仓库：

```bash
make docker-push REGISTRY=<your-registry>
```

**示例**:

```bash
# 推送到 GitHub Container Registry
make docker-push REGISTRY=ghcr.io/your-org

# 推送到 Docker Hub
make docker-push REGISTRY=docker.io/your-namespace
```

### 5. 镜像内部结构与配置

**工作目录**: `/home/work/api-server`

**目录结构**:
```
/home/work/api-server/
├── api-server          # 服务二进制
├── conf/               # 配置目录（可通过 volume 覆盖）
├── static/             # 静态资源（Dashboard 前端）
└── log/                # 日志目录
```

**推荐配置挂载**:

```bash
docker run -d \
  --name ai-gateway-api \
  -p 8183:8183 \
  -v $(pwd)/conf:/home/work/api-server/conf \
  -v $(pwd)/log:/home/work/api-server/log \
  ai-gateway-api:latest
```

**配置文件说明**:
- `conf/` 目录包含数据库连接、Redis 配置、鉴权配置等
- 通过挂载本地配置目录，可实现配置热更新而无需重建镜像

---

## Service Controller 编译

Service Controller 负责 Kubernetes 服务发现，自动将 Service 资源注册到 BFE 配置中。

### 1. 源码获取

**GitHub 仓库**: [https://github.com/bfenetworks/service-controller](https://github.com/bfenetworks/service-controller)

克隆源码：

```bash
git clone https://github.com/bfenetworks/service-controller.git
cd service-controller
```

### 2. 配置 Go 代理（国内网络强烈推荐）

Service Controller 的编译过程需要下载较多 Go 依赖，**强烈建议**配置 Go 代理：

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.cn,direct

# 预下载依赖
go mod download
```

### 3. 编译二进制（可选）

如需本地运行或调试，可先编译二进制：

```bash
make build
```

编译产物位于 `./bin/` 目录。

### 4. 本地镜像构建

在仓库根目录执行：

```bash
make docker
```

**镜像特点**:
- **基础镜像**: Alpine Linux（轻量级、安全性高）
- **多架构支持**: 同时支持 x86_64 和 ARM64
- **镜像体积小**: 得益于 Alpine 基础镜像和 Go 静态编译

**构建产物**:
```
service-controller:latest
```

### 5. 验证镜像

#### 查看构建的镜像

```bash
docker images | grep service-controller
```

#### 在 Kubernetes 中验证

Service Controller 设计为在 Kubernetes 集群中运行，本地验证建议直接部署到集群：

```bash
# 应用部署清单（需提前配置 BFE API Server 地址）
kubectl apply -f ./examples/service-controller-endpoints.yaml

# 检查部署状态
kubectl get deployment bfe-service-controller
kubectl get pods | grep service-controller

# 查看日志
kubectl logs -f <pod-name>
```

**健康检查端点**:
- Readiness: `GET /ready`
- Liveness: `GET /healthz`

### 6. 多架构镜像推送

推送到远程仓库供 Kubernetes 集群使用：

```bash
make docker-push REGISTRY=<your-registry>
```

**示例**:

```bash
# 推送到 GitHub Container Registry（多架构）
make docker-push REGISTRY=ghcr.io/your-org

# 推送到私有仓库
make docker-push REGISTRY=registry.example.com/team
```

**推送的镜像**:
```
ghcr.io/your-org/service-controller:latest
```

### 7. Kubernetes 部署配置要点

**前置依赖**:
- Kubernetes 集群版本 >= v1.18
- BFE API Server 已部署并可访问

**关键配置** (`examples/service-controller-endpoints.yaml`):
- `bfe-api-addr`: BFE API Server 地址
- `bfe-api-token`: API Server 认证 Token（从 Dashboard 获取）
- `namespace`: 监听的 Kubernetes 命名空间

**Service 注解要求**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
  labels:
    bfe-product: demo  # 必须：指定 BFE 产品线
spec:
  ports:
    - name: http       # 必须：端口必须命名
      port: 8080
      targetPort: 80
```

---

## Kubernetes 部署集成

本章节演示如何将上述编译的三个组件镜像集成部署到 Kubernetes 集群中。

### 1. 前置准备

#### 确认集群环境

```bash
# 检查 kubectl 配置
kubectl cluster-info

# 检查节点状态
kubectl get nodes
```

#### 创建命名空间

按照项目规范，使用独立命名空间隔离资源：

```bash
kubectl apply -f kubernetes/namespace.yaml
```

验证命名空间：
```bash
kubectl get namespace ai-gateway-demo
```

### 2. 更新镜像引用

根据您推送的镜像地址，修改 `kubernetes/kustomization.yaml` 中的镜像配置：

```yaml
images:
  - name: bfenetworks/bfe
    newName: ghcr.io/your-org/bfe        # 替换为您的镜像地址
    newTag: v1.8.0                        # 替换为您的版本
  - name: bfenetworks/service-controller
    newName: ghcr.io/your-org/service-controller
    newTag: latest
  - name: ai-gateway-api
    newName: ghcr.io/your-org/ai-gateway-api
    newTag: latest
  - name: traefik/whoami
    newName: traefik/whoami               # 示例应用，可保持不变
    newTag: latest
```

### 3. 镜像拉取凭证（私有仓库）

如果您的镜像存储在私有仓库，需要创建 `imagePullSecrets`：

```bash
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=<your-username> \
  --docker-password=<your-token> \
  --namespace=ai-gateway-demo
```

在部署清单中引用凭证（修改 `*-deploy.yaml`）：

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-secret
```

### 4. 部署基础依赖

#### 部署 MySQL

```bash
kubectl apply -f kubernetes/mysql-deploy.yaml
```

验证：
```bash
kubectl get pods -n ai-gateway-demo | grep mysql
kubectl logs -n ai-gateway-demo <mysql-pod-name>
```

#### 部署 Redis

```bash
kubectl apply -f kubernetes/redis-deploy.yaml
```

验证：
```bash
kubectl get pods -n ai-gateway-demo | grep redis
```

### 5. 部署核心服务

#### 部署 AI Gateway API（控制面）

```bash
# 应用 ConfigMap（如需自定义配置）
kubectl apply -f kubernetes/ai-gateway-configmap.yaml

# 部署服务
kubectl apply -f kubernetes/service-controller-deploy.yaml
```

验证：
```bash
# 检查 Pod 状态
kubectl get pods -n ai-gateway-demo -l app=ai-gateway-api

# 检查日志
kubectl logs -n ai-gateway-demo -l app=ai-gateway-api -f

# 端口转发测试（可选）
kubectl port-forward -n ai-gateway-demo svc/ai-gateway-api 8183:8183
# 访问 http://localhost:8183
```

#### 部署 BFE（数据面）

```bash
# 应用 ConfigMap
kubectl apply -f kubernetes/bfe-configmap.yaml

# 部署服务
kubectl apply -f kubernetes/bfe-deploy.yaml
```

验证：
```bash
# 检查 Pod 状态
kubectl get pods -n ai-gateway-demo -l app=bfe

# 检查日志
kubectl logs -n ai-gateway-demo -l app=bfe -f

# 检查服务端口
kubectl get svc -n ai-gateway-demo bfe
```

#### 部署 Service Controller

```bash
kubectl apply -f kubernetes/service-controller-deploy.yaml
```

验证：
```bash
# 检查 Pod 状态
kubectl get pods -n ai-gateway-demo -l app=service-controller

# 检查日志（确认已连接 BFE API Server）
kubectl logs -n ai-gateway-demo -l app=service-controller -f
```

### 6. 部署测试应用

部署 whoami 应用用于验证流量转发：

```bash
kubectl apply -f kubernetes/whoami-deploy.yaml
```

验证服务自动注册：
```bash
# 检查 whoami 服务
kubectl get svc -n ai-gateway-demo whoami

# 检查 Service Controller 日志，应显示服务注册成功
kubectl logs -n ai-gateway-demo -l app=service-controller | grep whoami
```

### 7. 完整部署（一键部署）

使用 Kustomize 一次性部署所有资源：

```bash
kubectl apply -k kubernetes/
```

**说明**:
- `-k` 参数指定使用 Kustomize 目录
- 自动应用 `kustomization.yaml` 中的镜像替换和资源顺序
- 确保在执行前已正确配置镜像引用

### 8. 验证完整部署

#### 检查所有 Pod 状态

```bash
kubectl get pods -n ai-gateway-demo
```

预期输出：
```
NAME                                READY   STATUS    RESTARTS   AGE
ai-gateway-api-xxx                  1/1     Running   0          5m
bfe-xxx                             1/1     Running   0          5m
mysql-xxx                           1/1     Running   0          6m
redis-xxx                           1/1     Running   0          6m
service-controller-xxx              1/1     Running   0          5m
whoami-xxx                          1/1     Running   0          4m
```

#### 测试端到端流量

```bash
# 获取 BFE Service 的 ClusterIP
BFE_IP=$(kubectl get svc bfe -n ai-gateway-demo -o jsonpath='{.spec.clusterIP}')

# 从集群内测试（创建测试 Pod）
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -n ai-gateway-demo -- \
  curl http://$BFE_IP:8080/

# 或通过端口转发测试
kubectl port-forward -n ai-gateway-demo svc/bfe 8080:8080
# 本地访问 http://localhost:8080
```

#### 访问控制面 Dashboard

```bash
# 端口转发 AI Gateway API
kubectl port-forward -n ai-gateway-demo svc/ai-gateway-api 8183:8183
```

浏览器访问: http://localhost:8183

---

## 完整示例

### 端到端演示流程

本节通过一个完整示例，演示从源码编译到生产部署的全流程。

#### 场景说明

假设您需要：
1. 编译最新的 BFE v1.8.0、AI Gateway API 和 Service Controller
2. 推送镜像到 GitHub Container Registry
3. 部署到生产 Kubernetes 集群（命名空间隔离）

#### Step 1: 编译所有组件

```bash
# 设置 Go 代理
export GOPROXY=https://goproxy.cn,direct

# 编译 BFE
cd /path/to/bfe
git checkout v1.8.0
make docker
make docker-push REGISTRY=ghcr.io/your-org

# 编译 AI Gateway API
cd /path/to/ai-gateway-api
git pull origin main
make docker DASHBOARD_VERSION=v0.0.1
make docker-push REGISTRY=ghcr.io/your-org

# 编译 Service Controller
cd /path/to/service-controller
git pull origin main
make docker
make docker-push REGISTRY=ghcr.io/your-org
```

#### Step 2: 准备 Kubernetes 环境

```bash
# 切换到项目目录
cd /path/to/ai-gateway-demo

# 创建命名空间
kubectl apply -f kubernetes/namespace.yaml

# 创建镜像拉取凭证
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=your-username \
  --docker-password=$GITHUB_TOKEN \
  --namespace=ai-gateway-demo
```

#### Step 3: 更新镜像配置

编辑 `kubernetes/kustomization.yaml`：

```yaml
images:
  - name: bfenetworks/bfe
    newName: ghcr.io/your-org/bfe
    newTag: v1.8.0
  - name: bfenetworks/service-controller
    newName: ghcr.io/your-org/service-controller
    newTag: latest
  - name: ai-gateway-api
    newName: ghcr.io/your-org/ai-gateway-api
    newTag: latest
```

#### Step 4: 一键部署

```bash
kubectl apply -k kubernetes/
```

#### Step 5: 验证部署

```bash
# 等待所有 Pod 就绪
kubectl wait --for=condition=ready pod -l app -n ai-gateway-demo --timeout=300s

# 检查状态
kubectl get pods -n ai-gateway-demo

# 测试流量转发
kubectl port-forward -n ai-gateway-demo svc/bfe 8080:8080 &
curl http://localhost:8080/

# 访问控制台
kubectl port-forward -n ai-gateway-demo svc/ai-gateway-api 8183:8183 &
# 浏览器访问 http://localhost:8183
```

---

## 故障排查

### 1. 镜像拉取失败

**现象**:
```
kubectl get pods -n ai-gateway-demo
# STATUS: ImagePullBackOff / ErrImagePull
```

**排查步骤**:

1. 检查镜像地址是否正确：
   ```bash
   kubectl describe pod <pod-name> -n ai-gateway-demo | grep Image
   ```

2. 验证镜像是否存在：
   ```bash
   docker pull ghcr.io/your-org/bfe:v1.8.0
   ```

3. 检查 imagePullSecrets 配置（私有仓库）：
   ```bash
   kubectl get secret ghcr-secret -n ai-gateway-demo
   kubectl describe pod <pod-name> -n ai-gateway-demo | grep ImagePullSecrets
   ```

4. 验证凭证是否有效：
   ```bash
   kubectl delete secret ghcr-secret -n ai-gateway-demo
   kubectl create secret docker-registry ghcr-secret \
     --docker-server=ghcr.io \
     --docker-username=<username> \
     --docker-password=<token> \
     --namespace=ai-gateway-demo
   ```

### 2. Pod 启动失败（CrashLoopBackOff）

**排查步骤**:

1. 查看 Pod 日志：
   ```bash
   kubectl logs <pod-name> -n ai-gateway-demo
   kubectl logs <pod-name> -n ai-gateway-demo --previous  # 查看上次崩溃日志
   ```

2. 检查 Pod 事件：
   ```bash
   kubectl describe pod <pod-name> -n ai-gateway-demo
   ```

3. 常见原因：
   - **配置错误**: 检查 ConfigMap 是否正确挂载
   - **依赖未就绪**: 确保 MySQL/Redis 先于应用启动
   - **资源不足**: 检查节点资源 `kubectl describe node`
   - **健康检查失败**: 调整 readinessProbe/livenessProbe 参数

### 3. Service Controller 无法注册服务

**现象**: Service Controller 日志显示连接 BFE API Server 失败

**排查步骤**:

1. 检查 AI Gateway API 是否正常运行：
   ```bash
   kubectl get pods -n ai-gateway-demo -l app=ai-gateway-api
   kubectl logs -n ai-gateway-demo -l app=ai-gateway-api
   ```

2. 验证 Service Controller 配置：
   ```bash
   kubectl describe deployment service-controller -n ai-gateway-demo | grep -A 10 "Environment"
   ```

3. 检查网络连通性：
   ```bash
   kubectl exec -it <service-controller-pod> -n ai-gateway-demo -- \
     wget -qO- http://ai-gateway-api:8183/healthz
   ```

4. 验证 API Token（如配置）：
   - 从 Dashboard 获取有效 Token
   - 更新 Deployment 环境变量或 Secret

### 4. BFE 流量转发异常

**排查步骤**:

1. 检查 BFE 配置加载：
   ```bash
   kubectl logs -n ai-gateway-demo -l app=bfe | grep "conf load"
   ```

2. 验证路由配置：
   ```bash
   # 访问 BFE 监控端点
   kubectl port-forward -n ai-gateway-demo svc/bfe 8421:8421
   # 浏览器访问 http://localhost:8421/monitor/product_status
   ```

3. 检查 Service 标签：
   ```bash
   kubectl get svc whoami -n ai-gateway-demo -o yaml | grep "bfe-product"
   ```
   
   确保 Service 包含必需标签 `bfe-product: <product-name>`

4. 检查 Endpoint 是否正常：
   ```bash
   kubectl get endpoints whoami -n ai-gateway-demo
   ```

### 5. 数据库连接失败

**排查步骤**:

1. 检查 MySQL Pod 状态：
   ```bash
   kubectl get pods -n ai-gateway-demo | grep mysql
   kubectl logs -n ai-gateway-demo <mysql-pod>
   ```

2. 验证 Service 可达性：
   ```bash
   kubectl exec -it <api-pod> -n ai-gateway-demo -- \
     nc -zv mysql 3306
   ```

3. 检查数据库初始化：
   ```bash
   kubectl exec -it <mysql-pod> -n ai-gateway-demo -- \
     mysql -u root -p<password> -e "SHOW DATABASES;"
   ```

### 6. 日志收集与诊断

**收集所有组件日志**:

```bash
# 创建日志目录
mkdir -p /tmp/ai-gateway-logs

# 收集各组件日志
kubectl logs -n ai-gateway-demo -l app=bfe > /tmp/ai-gateway-logs/bfe.log
kubectl logs -n ai-gateway-demo -l app=ai-gateway-api > /tmp/ai-gateway-logs/api.log
kubectl logs -n ai-gateway-demo -l app=service-controller > /tmp/ai-gateway-logs/controller.log
kubectl logs -n ai-gateway-demo -l app=mysql > /tmp/ai-gateway-logs/mysql.log
kubectl logs -n ai-gateway-demo -l app=redis > /tmp/ai-gateway-logs/redis.log

# 收集事件
kubectl get events -n ai-gateway-demo --sort-by='.lastTimestamp' > /tmp/ai-gateway-logs/events.log
```

**查看实时日志**:
```bash
kubectl logs -f -n ai-gateway-demo -l app=bfe --all-containers=true --tail=100
```

---

## 参考资料

### 官方文档

- **BFE 项目**
  - GitHub: https://github.com/bfenetworks/bfe
  - 官方文档: https://www.bfe-networks.net/
  - 配置参考: https://www.bfe-networks.net/en_us/configuration/overview/

- **AI Gateway API**
  - GitHub: https://github.com/yf-networks/ai-gateway-api
  - Dashboard 前端: https://github.com/yf-networks/ai-gateway-web

- **Service Controller**
  - GitHub: https://github.com/bfenetworks/service-controller
  - Kubernetes 集成指南: https://github.com/bfenetworks/service-controller/blob/main/README.md

### 技术栈文档

- **Kubernetes**
  - 官方文档: https://kubernetes.io/docs/
  - Kustomize: https://kubectl.docs.kubernetes.io/references/kustomize/

- **Docker**
  - Docker Buildx: https://docs.docker.com/buildx/working-with-buildx/
  - 多架构构建: https://docs.docker.com/build/building/multi-platform/

- **Go 语言**
  - 官方网站: https://go.dev/
  - GOPROXY 镜像: https://goproxy.cn/

### 镜像仓库

- **GitHub Container Registry**
  - 文档: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
  - 认证指南: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-to-the-container-registry

- **Docker Hub**
  - 官方网站: https://hub.docker.com/
  - 推送镜像指南: https://docs.docker.com/docker-hub/repos/

### 本项目资源

- **项目宪章**: `.specify/memory/constitution.md`
- **功能规范**: `specs/001-build-guide/spec.md`
- **实施计划**: `specs/001-build-guide/plan.md`
- **任务清单**: `specs/001-build-guide/tasks.md`

### 社区与支持

- **BFE 社区**
  - 邮件列表: https://lists.cncf.io/g/bfe-dev
  - Slack: https://cloud-native.slack.com/messages/bfe

- **问题反馈**
  - BFE Issues: https://github.com/bfenetworks/bfe/issues
  - Service Controller Issues: https://github.com/bfenetworks/service-controller/issues
  - AI Gateway API Issues: https://github.com/yf-networks/ai-gateway-api/issues

---

## 贡献与反馈

我们非常欢迎您对本文档提出改进建议或报告问题。

### 报告文档问题

如果您在使用本指南过程中发现以下问题，请通过 GitHub Issue 向我们反馈：

- **技术错误**: 命令执行失败、参数配置错误
- **内容遗漏**: 缺少关键步骤或必要说明
- **描述不清**: 说明模糊、难以理解的部分
- **链接失效**: 无法访问的外部资源链接

**提交 Issue 步骤**:

1. 访问项目仓库: https://github.com/cc14514/ai-gateway-demo
2. 进入 **Issues** 标签页
3. 点击 **New Issue** 创建新问题
4. 使用标题格式: `[BUILD-GUIDE] 简短问题描述`
5. 在描述中包含:
   - 问题所在章节（如 "BFE 编译 - 镜像构建"）
   - 详细的问题描述
   - 如为命令错误，请附上完整的错误日志
   - 您的操作系统和工具版本信息

### 提交改进建议

如果您有以下改进想法，欢迎通过 GitHub Issue 或 Pull Request 贡献：

- **补充最佳实践**: 您在实践中总结的优化方案
- **增加示例场景**: 更多实际应用场景的配置示例
- **优化表述**: 让说明更清晰易懂的改写建议
- **新增章节**: 您认为应该补充的内容

**提交 Pull Request 步骤**:

1. Fork 项目仓库到您的 GitHub 账号
2. 克隆您的 Fork 仓库到本地
3. 创建特性分支: `git checkout -b feature/improve-build-guide`
4. 编辑 BUILD-GUIDE-CN.md 文件
5. 提交更改: `git commit -s -m "docs: 改进 BFE 编译章节说明"`
6. 推送到您的 Fork: `git push origin feature/improve-build-guide`
7. 在 GitHub 上创建 Pull Request
8. 在 PR 描述中说明您的改进内容和原因

### 文档维护原则

在贡献时，请遵循以下原则（详见项目宪章 `.specify/memory/constitution.md`）：

- **Documentation-First**: 保持中英文双语支持的规划
- **Demo-Focused Clarity**: 确保说明清晰、步骤可验证
- **Production-Ready Warnings**: 明确标注生产环境注意事项
- **Namespace Isolation**: 示例命令使用独立命名空间

### 联系方式

- **项目仓库**: https://github.com/cc14514/ai-gateway-demo
- **Issue 追踪**: https://github.com/cc14514/ai-gateway-demo/issues
- **贡献指南**: 参考项目根目录 CONTRIBUTING.md（如有）

---
