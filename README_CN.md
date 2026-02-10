[English](./README.md) | 简体中文

# AI Gateway Kubernetes 部署示例

## 概述

![BFE Kubernetes](./.images/ai-gateway-k8s.png)

本示例在 `ai-gateway-system` 命名空间中演示了若干关键组件及其交互：
- 数据面（bfe 与 conf-agent）负责流量转发与接入控制；
- 控制面（ai-gateway-api）负责策略/配置下发接口；
- 基础依赖（MySQL、Redis）为控制面提供存储与依赖服务；
- 服务发现（service-controller）负责发现并同步后端服务；
- 示例服务 whoami 用于验证路由；
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
| `whoami-deploy.yaml` | 示例测试服务 whoami 的 Deployment 清单 |

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
kubectl apply -f kubernetes/whoami-deploy.yaml
```

> whoami 部署在 `default` 命名空间；如需替换镜像，请直接修改 `whoami-deploy.yaml`。

### 4) 快速验证

```bash
kubectl get pods -n ai-gateway-system
kubectl get svc -n ai-gateway-system
```

访问 Dashboard（默认账号/密码：admin/admin）：

```
http://{NodeIP}:30183
```

## 常见操作

### 清理部署

```bash
kubectl delete -f kubernetes/whoami-deploy.yaml
kubectl delete -k kubernetes/
```

> 建议先删 whoami 再删 `ai-gateway-system`，避免 finalizers 导致卡住。

## 提交 Issue

如果遇到问题或有改进建议，请提交 Issue：

- 入口： https://github.com/yf-networks/ai-gateway-demo/issues
- 建议包含：环境信息、复现步骤、报错日志、期望行为

## 参考资料

- 构建与部署完整指南： [BUILD_GUIDE_CN.md](./BUILD_GUIDE_CN.md)
- BFE 项目： https://github.com/bfenetworks/bfe
- AI Gateway API： https://github.com/yf-networks/ai-gateway-api
- Service Controller： https://github.com/bfenetworks/service-controller
- Dashboard 前端： https://github.com/yf-networks/ai-gateway-web
- Kubernetes 文档： https://kubernetes.io/docs/


---------

## 贡献与反馈

我们非常欢迎您对本文档提出改进建议或报告问题。

- BFE Issues: https://github.com/bfenetworks/bfe/issues
- Service Controller Issues: https://github.com/bfenetworks/service-controller/issues
- AI Gateway API Issues: https://github.com/yf-networks/ai-gateway-api/issues

### 报告文档问题

如果您在使用本指南过程中发现以下问题，请通过 GitHub Issue 向我们反馈：

- **技术错误**: 命令执行失败、参数配置错误
- **内容遗漏**: 缺少关键步骤或必要说明
- **描述不清**: 说明模糊、难以理解的部分
- **链接失效**: 无法访问的外部资源链接

**提交 Issue 步骤**:

1. 访问项目仓库: https://github.com/yf-networks/ai-gateway-demo
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
4. 编辑 BUILD_GUIDE_CN.md 文件（假设）
5. 提交更改: `git commit -s -m "docs: 改进 BFE 编译章节说明"`
6. 推送到您的 Fork: `git push origin feature/improve-build-guide`
7. 在 GitHub 上创建 Pull Request
8. 在 PR 描述中说明您的改进内容和原因

### 文档维护原则

在贡献时，请遵循以下原则：

- **Documentation-First**: 保持中英文双语支持的规划
- **Demo-Focused Clarity**: 确保说明清晰、步骤可验证
- **Production-Ready Warnings**: 明确标注生产环境注意事项

### 联系方式

- **项目仓库**: https://github.com/yf-networks/ai-gateway-demo
- **Issue 追踪**: https://github.com/yf-networks/ai-gateway-demo/issues
- **电子邮件**: liangchuan@yf-networks.com 
