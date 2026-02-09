# msgapp-helm

Helm Chart for msgapp - A real-time messaging application with GitOps deployment.

## Architecture

- **Frontend**: Next.js application
- **Backend**: Node.js/Express with Socket.io
- **Worker**: Async message processor
- **Redis**: Caching layer for real-time messaging
- **RabbitMQ**: Message queue for database writes
- **MongoDB**: Persistent data storage

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
