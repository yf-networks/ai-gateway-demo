English | [简体中文](./README_CN.md)

# AI Gateway Kubernetes Deployment Example

## Overview

![BFE Kubernetes](./.images/ai-gateway-k8s.png)

This example demonstrates the interaction of several key components in the `ai-gateway-system` namespace:
- Data plane (bfe with conf-agent): handles traffic forwarding and access control
- Control plane (ai-gateway-api): provides configuration/policy delivery API
- Base dependencies (MySQL, Redis): provide storage and dependency services for the control plane
- Service discovery (service-controller): discovers and syncs backend services
- Demo service (whoami): used to validate routing
- Components communicate via Kubernetes Service/DNS, for example:
  - ai-gateway-api.ai-gateway-system.svc.cluster.local
  - mysql.ai-gateway-system.svc.cluster.local
  - redis.ai-gateway-system.svc.cluster.local

Notes:
- MySQL / Redis use `emptyDir` for storage in this example and data will be lost on Pod restart
- This example is for demo/connectivity validation and is not production-ready

📖 **[Build Guide](./BUILD_GUIDE.md)**: Complete guide from source compilation to Docker image building and Kubernetes deployment

Main files (directory ./kubernetes):

| **File** | **Description** |
|---|---|
| `namespace.yaml` | Namespace definition (ai-gateway-system) |
| `kustomization.yaml` | Kustomize resource aggregation and enable/disable options |
| `bfe-configmap.yaml` | bfe configuration (bfe.conf, conf-agent.toml, etc.) |
| `bfe-deploy.yaml` | bfe data plane Deployment manifest |
| `ai-gateway-configmap.yaml` | ai-gateway-api configuration (DB/Redis connection, auth example) |
| `ai-gateway-deploy.yaml` | ai-gateway-api Deployment/Service manifest |
| `mysql-deploy.yaml` | MySQL Deployment (example database and storage config) |
| `redis-deploy.yaml` | Redis Deployment/Service (example cache config) |
| `service-controller-deploy.yaml` | Service discovery controller Deployment manifest |
| `whoami-deploy.yaml` | whoami demo service Deployment manifest |

## Prerequisites

- kubectl with `-k` support (recommended kubectl >= 1.20)
- kubectl can access the target cluster with permissions to create Namespace, Deployment, Service, ConfigMap, Secret
- Cluster nodes can pull images (configure image acceleration or private registry credentials if needed)

## Quick Start

This README provides only the shortest path to get started. For complete source compilation, image building, and Kubernetes deployment, see:
📖 **[Build Guide](./BUILD_GUIDE.md)**

### 1) Configure Images (Optional)

To replace image addresses or versions, modify `images:` in `kustomization.yaml`:

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

### 2) One-Command Deployment

```bash
kubectl apply -k kubernetes 
```

This command deploys: bfe (with conf-agent), ai-gateway-api (with Dashboard), mysql, redis, service-controller.

### 3) Deploy Test Service (Optional)

```bash
kubectl apply -f kubernetes/whoami-deploy.yaml
```

> whoami is deployed in the `default` namespace; to replace the image, edit `whoami-deploy.yaml` directly.

### 4) Quick Validation

```bash
kubectl get pods -n ai-gateway-system
kubectl get svc -n ai-gateway-system
```

Access Dashboard (default username/password: admin/admin):

```
http://{NodeIP}:30183
```

## Common Operations

### Cleanup Deployment

```bash
kubectl delete -f kubernetes/whoami-deploy.yaml
kubectl delete -k kubernetes/
```

> Recommended: delete whoami first, then `ai-gateway-system` to avoid finalizers causing hang.

## Submit Issues

If you encounter problems or have suggestions:

- Entry: https://github.com/yf-networks/ai-gateway-demo/issues
- Please include: environment info, reproduction steps, error logs, expected behavior

## References

- Build and Deployment Complete Guide: [BUILD_GUIDE.md](./BUILD_GUIDE.md) / [BUILD_GUIDE_CN.md](./BUILD_GUIDE_CN.md)
- BFE Project: https://github.com/bfenetworks/bfe
- AI Gateway API: https://github.com/yf-networks/ai-gateway-api
- Service Controller: https://github.com/bfenetworks/service-controller
- Dashboard Frontend: https://github.com/yf-networks/ai-gateway-web
- Kubernetes Documentation: https://kubernetes.io/docs/


---------

## Contributions and Feedback

We welcome improvements and issue reports for this document.

- BFE Issues: https://github.com/bfenetworks/bfe/issues
- Service Controller Issues: https://github.com/bfenetworks/service-controller/issues
- AI Gateway API Issues: https://github.com/yf-networks/ai-gateway-api/issues

### Report Documentation Issues

If you encounter any of the following, please report via GitHub Issue:

- **Technical errors**: command failures, incorrect parameters
- **Missing content**: missing steps or required details
- **Unclear descriptions**: ambiguous or hard-to-understand text
- **Broken links**: inaccessible external references

**Steps to submit an Issue**:

1. Visit the repository: https://github.com/yf-networks/ai-gateway-demo
2. Go to the **Issues** tab
3. Click **New Issue**
4. Use the title format: `[BUILD-GUIDE] Short issue description`
5. Include:
  - The section where the issue appears (e.g., "BFE Build - Image Build")
  - Detailed description
  - Full error logs (if applicable)
  - Your OS and tool versions

### Submit Improvements

If you have ideas to improve the guide, please contribute via Issue or Pull Request:

- **Add best practices**: optimizations you discovered in practice
- **Add example scenarios**: more real-world configuration examples
- **Improve wording**: clearer and more concise explanations
- **Add new sections**: content you think should be included

**Steps to submit a Pull Request**:

1. Fork the repository to your GitHub account
2. Clone your fork locally
3. Create a feature branch: `git checkout -b feature/improve-build-guide`
4. Edit BUILD_GUIDE.md / BUILD_GUIDE_CN.md (as needed)
5. Commit changes: `git commit -s -m "docs: improve build guide"`
6. Push to your fork: `git push origin feature/improve-build-guide`
7. Create a Pull Request on GitHub
8. Describe what you changed and why in the PR

### Documentation Principles

Please follow these principles when contributing:

- **Documentation-First**: keep bilingual support in mind
- **Demo-Focused Clarity**: keep steps clear and verifiable
- **Production-Ready Warnings**: clearly mark production considerations

### Contact

- **Repository**: https://github.com/yf-networks/ai-gateway-demo
- **Issue Tracker**: https://github.com/yf-networks/ai-gateway-demo/issues
- **Email**: liangchuan@yf-networks.com 
