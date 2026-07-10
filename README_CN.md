[English](./README.md) | 简体中文

# AI Gateway Kubernetes 部署示例

> **本仓库已被 [ai-gateway](https://github.com/yf-networks/ai-gateway) 替代。**
> 新的 `ai-gateway` 仓库提供 all-in-one Docker 部署方案（`docker compose up -d`），支持数据库自动初始化、Docker 网络 DNS 和测试模拟器 — 无需 Kubernetes。
>
> 最新版本及容器部署指南见 👉 [README.md](https://github.com/yf-networks/ai-gateway/blob/main/README.md)
>
> 本仓库的 K8s 部署清单已迁移至 `ai-gateway/kubernetes/`。

## 概述

![BFE Kubernetes](./.images/ai-gateway-k8s.png)

本示例在 `ai-gateway-system` 命名空间中演示了若干关键组件及其交互：
- 数据面（bfe 与 conf-agent）负责流量转发与接入控制；
- 控制面（ai-gateway-api）负责策略/配置下发接口；
- 基础依赖（MySQL、Redis）为控制面提供存储与依赖服务；
- 服务发现（service-controller）负责发现并同步后端服务；
- 示例后端服务（llm-d inference simulator）用于验证路由；
- 组件间通过 Kubernetes Service/DNS 相互通信，如：
  - ai-gateway-api.ai-gateway-system.svc.cluster.local
  - mysql.ai-gateway-system.svc.cluster.local
  - redis.ai-gateway-system.svc.cluster.local

注意：
- 示例中的 MySQL / Redis 使用 `emptyDir` 作为存储，会随 Pod 重启丢失数据；
- 本示例偏向演示与联通性验证，不能直接用于生产环境。

📖 **[编译指南](./BUILD_GUIDE_CN.md)**：从源代码编译到 Docker 镜像构建再到 Kubernetes 部署的完整指南

主要文件概览(kubernetes 目录)：

| **文件名** | **说明** |
|---|---|
| `namespace.yaml` | 命名空间定义（ai-gateway-system） |
| `kustomization.yaml` | kustomize 资源汇总与启用/禁用项 |
| `bfe-configmap.yaml` | bfe 配置（bfe.conf、conf-agent.toml 等） |
| `bfe-deploy.yaml` | bfe 数据面 Deployment 清单 |
| `ai-gateway-configmap.yaml` | ai-gateway-api 配置（DB/Redis 连接、鉴权示例） |
| `ai-gateway-deploy.yaml` | ai-gateway-api Deployment/Service 清单 |
| `mysql-deploy.yaml` | MySQL Deployment（示例数据库与存储配置） |
| `redis-deploy.yaml` | Redis Deployment/Service（示例缓存配置） |
| `service-controller-deploy.yaml` | 服务发现控制器 Deployment 清单 |
| `llm-d-inference-sim-deploy.yaml` | 示例后端服务（llm-d inference simulator）清单 |

## 前提条件

- kubectl 支持 `-k`（建议 kubectl >= 1.20）
- kubectl 可访问目标集群且具备创建 Namespace、Deployment、Service、ConfigMap、Secret 的权限
- 集群节点可拉取镜像（必要时配置镜像加速或私有仓库凭证）

## 快速开始

本 README 仅提供最短路径的快速上手，完整的源码编译、镜像构建与 Kubernetes 部署请参见：
📖 **[编译指南](./BUILD_GUIDE_CN.md)**

### 1) 配置镜像（可选）

如需替换镜像地址或版本，统一在 `kustomization.yaml` 的 `images:` 中修改：

```yaml
images:
  - name: bfenetworks/bfe
    newName: ghcr.io/your-org/bfe
    newTag: v1.8.0
  - name: ai-gateway-api
    newName: ghcr.io/your-org/ai-gateway-api
    newTag: latest
  - name: bfenetworks/service-controller
    newName: ghcr.io/your-org/service-controller
    newTag: latest
```

### 2) 一键部署核心组件

```bash
kubectl apply -k kubernetes 
```

该命令会部署：bfe（含 conf-agent）、ai-gateway-api（含 Dashboard）、mysql、redis、service-controller。

### 3) 部署测试服务（可选）

```bash
kubectl apply -f kubernetes/llm-d-inference-sim-deploy.yaml
```

> 示例后端服务部署在 `default` 命名空间；如需替换镜像/模型参数，请直接修改 `llm-d-inference-sim-deploy.yaml`。

### 4) 快速验证

```bash
kubectl get pods -n ai-gateway-system
kubectl get svc -n ai-gateway-system
```

访问 Dashboard（默认账号/密码：admin/admin）：

```
http://{NodeIP}:30183
```

**关于 Dashboard 的配置说明**

- 请参见：[ai-gateway-web](https://github.com/yf-networks/ai-gateway-web/blob/develop/README.md) 的 README


## 常见操作

### 清理部署

```bash
kubectl delete -f kubernetes/llm-d-inference-sim-deploy.yaml
kubectl delete -k kubernetes/
```

> 建议先删示例后端服务再删 `ai-gateway-system`，避免 finalizers 导致卡住。


## 社区贡献

贡献方式请见 [CONTRIBUTING.md](./CONTRIBUTING.md)。

## 参考资料

- 构建与部署完整指南： [BUILD_GUIDE_CN.md](./BUILD_GUIDE_CN.md)
- BFE 项目： https://github.com/bfenetworks/bfe
- AI Gateway API： https://github.com/yf-networks/ai-gateway-api
- 示例后端（llm-d inference simulator）： https://github.com/llm-d/llm-d-inference-sim
- Service Controller： https://github.com/bfenetworks/service-controller
- Dashboard 前端： https://github.com/yf-networks/ai-gateway-web
- Kubernetes 文档： https://kubernetes.io/docs/