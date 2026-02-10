English | [简体中文](./README-CN.md)

# AI Gateway Kubernetes Deployment Example

## Overview

![BFE Kubernetes](./.images/ai-gateway-k8s.png)

This example demonstrates several key components and their interactions in the `ai-gateway-system` namespace:
- Data plane (bfe with conf-agent): traffic forwarding and access control
- Control plane (ai-gateway-api): configuration/policy delivery API
- Base dependencies (MySQL, Redis): storage and dependency services for the control plane
- Service discovery (service-controller): discovers and syncs backend services
- Demo service (whoami): used to validate routing
- Components communicate via Kubernetes Service/DNS, for example:
  - ai-gateway-api.ai-gateway-system.svc.cluster.local
  - mysql.ai-gateway-system.svc.cluster.local
  - redis.ai-gateway-system.svc.cluster.local

Notes:
- MySQL / Redis use `emptyDir` storage in this example and data can be lost after Pod restarts.
- This is primarily for demo/connectivity validation and is not production-ready.

📖 **[Build Guide](./BUILD-GUIDE-CN.md)**: Complete guide from source compilation to Docker image building and Kubernetes deployment

Main files:

| **File** | **Description** |
|---|---|
| `namespace.yaml` | Namespace definition (ai-gateway-system) |
| `kustomization.yaml` | Kustomize resource aggregation and enable/disable options |
| `bfe-configmap.yaml` | BFE configuration (bfe.conf, conf-agent.toml, etc.) |
| `bfe-deploy.yaml` | BFE data plane Deployment manifest |
| `ai-gateway-configmap.yaml` | AI Gateway API configuration (DB/Redis connection, auth example) |
| `ai-gateway-deploy.yaml` | AI Gateway API Deployment/Service manifest |
| `mysql-deploy.yaml` | MySQL Deployment (demo database and storage config) |
| `redis-deploy.yaml` | Redis Deployment/Service (demo cache config) |
| `service-controller-deploy.yaml` | Service discovery controller Deployment manifest |
| `whoami-deploy.yaml` | whoami demo service Deployment manifest |

## Prerequisites

- kubectl with `-k` support (recommended kubectl >= 1.20)
- kubectl can access the target cluster with permissions to create Namespace, Deployment, Service, ConfigMap, Secret
- Cluster nodes can pull images (configure image acceleration or private registry credentials if needed)

## Quick Start

This README provides the shortest path to get started. For complete source compilation, image building, and Kubernetes deployment, see:
📖 **[Build Guide](./BUILD-GUIDE-CN.md)**

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
cd kubernetes
kubectl apply -k .
```

This deploys: bfe (with conf-agent), ai-gateway-api (with Dashboard), mysql, redis, service-controller.

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

- Build and Deployment Complete Guide: [BUILD-GUIDE-CN.md](./BUILD-GUIDE-CN.md)
- BFE Project: https://github.com/bfenetworks/bfe
- AI Gateway API: https://github.com/yf-networks/ai-gateway-api
- Service Controller: https://github.com/bfenetworks/service-controller
- Dashboard Frontend: https://github.com/yf-networks/ai-gateway-web
- Kubernetes Documentation: https://kubernetes.io/docs/

## Quick Start

This README provides the shortest path to get started. For complete source compilation, image building, and Kubernetes deployment, see:
📖 **[Build Guide](./BUILD-GUIDE-CN.md)**

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
cd kubernetes
kubectl apply -k .
```

This deploys: bfe (with conf-agent), ai-gateway-api (with Dashboard), mysql, redis, service-controller.

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

- Build and Deployment Complete Guide: [BUILD-GUIDE-CN.md](./BUILD-GUIDE-CN.md)
- BFE Project: https://github.com/bfenetworks/bfe
- AI Gateway API: https://github.com/yf-networks/ai-gateway-api
- Service Controller: https://github.com/bfenetworks/service-controller
- Dashboard Frontend: https://github.com/yf-networks/ai-gateway-web
- Kubernetes Documentation: https://kubernetes.io/docs/

To locate remaining resources (example):

```bash
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get -n ai-gateway-system --ignore-not-found --show-kind --no-headers
```

If you confirm it is safe to force cleanup, you can remove namespace finalizers (use with care):

```bash
kubectl patch ns ai-gateway-system --type=merge -p '{"spec":{"finalizers":[]}}'
```

- Delete individual resources (optional):

```bash
# bfe 
kubectl -n ai-gateway-system delete -f service-controller-deploy.yaml
kubectl -n ai-gateway-system delete -f ai-gateway-deploy.yaml
kubectl -n ai-gateway-system delete -f ai-gateway-configmap.yaml
kubectl -n ai-gateway-system delete -f mysql-deploy.yaml
kubectl -n ai-gateway-system delete -f redis-deploy.yaml
kubectl -n ai-gateway-system delete -f bfe-deploy.yaml
kubectl -n ai-gateway-system delete -f bfe-configmap.yaml

# whoami
kubectl delete -f whoami-deploy.yaml
```

## Restart

In this example, MySQL/Redis Pods use `emptyDir`, and data may be lost on restart.
If you want to keep the existing database while updating other Pods (e.g. after changing configs and running `kubectl apply -k .`), you can temporarily comment out or remove `mysql-deploy.yaml` (and optionally `redis-deploy.yaml`) from `kustomization.yaml`, then run `kubectl apply -k .`.

## Key configurations

### Data plane (bfe)

- Image: override via `images:` in `kustomization.yaml`. Use a pinned tag.

- Config mounts: `bfe-configmap.yaml` includes bfe.conf and conf-agent.toml. Make sure the mount paths match the paths referenced in your configs.

- Monitoring port: the example exposes 8421 for health/monitor endpoints. You can verify via Service or `kubectl port-forward`.

- Service ports: the example exposes 8080 (NodePort 30080) for external access.

### Control plane (ai-gateway-api and MySQL/Redis)

- DB connection: configure `Databases.bfe_db.Addr` in `ai_gateway_api.toml` inside `ai-gateway-configmap.yaml` (example: `mysql.ai-gateway-system.svc.cluster.local:3306`).
  - This example mixes plain text and Secret usage for passwords (MySQL root password in Secret + `Passwd` in toml). In production, use Secrets consistently and avoid plain text.

- Auth token: the example token is preconfigured in `bfe-configmap.yaml` (conf-agent.toml) and `service-controller-deploy.yaml`.
  - Use Secrets / short-lived / dynamically managed credentials in production.
  - In production, create tokens in the dashboard: https://github.com/yf-networks/ai-gateway-web/tree/develop

- MySQL storage: `mysql-deploy.yaml` uses an `emptyDir` volume for convenience. Data will be lost after Pod restart, not suitable for production.
  - In production, use PV/PVC with a StorageClass and a backup strategy.

- MySQL initialization: the example uses a Job to run `db_ddl.sql` to initialize schema. ai-gateway-api waits for tables in `open_bfe` via an initContainer before starting.
  - If startup is slow in your environment, increase `startupProbe` tolerances in `mysql-deploy.yaml` (e.g. bump `failureThreshold`).

- See also: [dashboard](https://github.com/yf-networks/ai-gateway-web/tree/develop)

### Service discovery (service-controller and whoami)

- Discovery rules: `service-controller-deploy.yaml` `args` / `env` define discovery strategy, label selectors, and API server address. Adjust to match your service labels/annotations.

- whoami ports: the container listens on port 80, while the Service exposes port 8080 (targetPort=80).

- See also: [service-controller](https://github.com/bfenetworks/service-controller/blob/main/README.md)

