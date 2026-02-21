# msgapp-helm

Helm Chart for msgapp - A real-time messaging application with GitOps deployment.

## Architecture

```mermaid
flowchart TD
    subgraph Internet ["🌐 Internet / User"]
        USER(["👤 Browser / Client"])
    end

    subgraph GitHub ["📦 GitHub — Source of Truth"]
        direction TB
        APP_REPO["App Repo\n(code + GitHub Actions CI)"]
        CHART_REPO["Helm Chart Repo\n1shailendra2/chart"]
    end

    subgraph ACR ["🐳 Azure Container Registry\nchatregi.azurecr.io"]
        IMG["msgapp-frontend/backend/worker:sha"]
    end

    subgraph AKS ["☸️ Azure Kubernetes Service"]

        subgraph argocd_ns ["namespace: argocd"]
            ARGOCD["🔄 ArgoCD\nApplication Controller"]
        end

        subgraph msgapp_ns ["namespace: msgapp-dev / msgapp-prod"]

            INGRESS["🚦 NGINX Ingress\n(sticky sessions via cookie)"]

            subgraph StatelessLayer ["Stateless — HPA-scaled"]
                FE["🖥️ Frontend\nNext.js · port 3000"]
                BE["⚙️ Backend\nNode.js + Socket.io · port 5000"]
                WK["🔧 Worker\nRabbitMQ consumer"]
            end

            subgraph StatefulLayer ["Stateful — PVC-backed"]
                REDIS["⚡ Redis\nport 6379"]
                RMQ["🐇 RabbitMQ\nport 5672 / 15672"]
                MONGO["🍃 MongoDB\nport 27017"]
            end

            K8S_SEC["🔑 Secret\nmsgapp-synced-secrets\n(JWT_SECRET, RABBITMQ_PASSWORD)"]
            BE_CFG["ConfigMap\nmsgapp-backend-config"]

            subgraph Monitoring ["📊 Monitoring"]
                PROM["Prometheus"]
                GRAFANA["Grafana"]
            end
        end
    end

    APP_REPO -- "build & push image" --> ACR
    APP_REPO -- "update image tag" --> CHART_REPO
    CHART_REPO -- "poll / webhook" --> ARGOCD
    ARGOCD -- "helm apply to namespace" --> msgapp_ns
    ACR -- "pull via acr-secret" --> StatelessLayer

    USER -- "HTTPS" --> INGRESS
    INGRESS -- "path: /" --> FE
    INGRESS -- "path: /api  /socket.io" --> BE
    FE -- "REST + WebSocket" --> BE
    BE -- "cache reads/writes" --> REDIS
    BE -- "publish message events" --> RMQ
    WK -- "consume messages" --> RMQ
    WK -- "persist to DB" --> MONGO
    BE -- "reads (via ConfigMap URL)" --> MONGO

    K8S_SEC -- "secretKeyRef" --> BE
    K8S_SEC -- "secretKeyRef" --> WK
    K8S_SEC -- "secretKeyRef" --> RMQ
    BE_CFG -- "envFrom" --> BE

    PROM -- "ServiceMonitor scrape :5000/metrics" --> BE
    PROM -- "ServiceMonitor scrape :5000/metrics" --> WK
    GRAFANA -- "PromQL" --> PROM
```

### Traffic Flow

| Step | What happens |
|------|-------------|
| 1 | User hits `msgapp.example.com` |
| 2 | NGINX Ingress checks path |
| 3a | `/` → Frontend pod (ClusterIP:3000), sticky cookie set |
| 3b | `/api` or `/socket.io` → Backend pod (ClusterIP:5000), same sticky cookie |
| 4 | Backend checks Redis cache; on miss reads MongoDB |
| 5 | On new message, Backend **publishes** to RabbitMQ |
| 6 | Worker **consumes** from RabbitMQ → persists to MongoDB |

### Components

| Component | Kind | Replicas (prod) | Notes |
|-----------|------|-----------------|-------|
| Frontend | Deployment | 3–20 (HPA) | Next.js, port 3000 |
| Backend | Deployment | 3–20 (HPA) | Node.js + Socket.io, port 5000 |
| Worker | Deployment | 2–10 (HPA) | No HTTP port, consumes RabbitMQ |
| Redis | Deployment | 1 | ephemeral `emptyDir`, AOF on |
| RabbitMQ | StatefulSet | 1 | 5Gi PVC |
| MongoDB | StatefulSet | 1 | 50Gi PVC in prod |

## Quick Start

### Prerequisites

- Kubernetes cluster
- Helm 3
- kubectl configured

### Install

```bash
# Clone this repository
git clone https://github.com/YOUR_USERNAME/msgapp-helm.git
cd msgapp-helm

# Install with default values
helm install msgapp ./helm-chart

# Install specific environment
helm install msgapp ./helm-chart \
  -f helm-chart/values.yaml \
  -f helm-chart/environments/values-prod.yaml \
  --namespace msgapp-prod \
  --create-namespace
```

### Upgrade

```bash
helm upgrade msgapp ./helm-chart

# With new image tag
helm upgrade msgapp ./helm-chart \
  --set backend.image.tag=abc123f \
  --set frontend.image.tag=abc123f \
  --set worker.image.tag=abc123f
```

## Configuration

See [values.yaml](helm-chart/values.yaml) for all configuration options.

## Environments

- **Development**: `environments/values-dev.yaml`
- **Staging**: `environments/values-staging.yaml`
- **Production**: `environments/values-prod.yaml`

## ArgoCD Deployment

```bash
# Deploy application
kubectl apply -f argocd/application-prod.yaml
```

## Documentation

- [Deployment Guide](helm-chart/DEPLOYMENT_GUIDE.md)
- [HPA Configuration](helm-chart/HPA.md)
- [Ingress Explained](helm-chart/INGRESS_EXPLAINED.md)

## License

MIT
