# GitOps Deployment Guide

## Architecture Overview

This guide covers deploying the msgapp using a GitOps workflow with:
- **2 Repositories**: Application code + Helm charts (Infrastructure as Code)
- **GitHub Actions**: CI/CD pipeline for building and pushing Docker images
- **Azure Container Registry**: Private Docker image registry
- **ArgoCD**: GitOps continuous deployment
- **Redis**: Caching layer for real-time messaging
- **RabbitMQ**: Message queue for async database writes
- **MongoDB**: Persistent data storage

## Prerequisites

- Azure account with ACR created
- Kubernetes cluster (AKS, GKE, or local)
- kubectl configured
- Helm 3 installed
- GitHub account

## Step 1: Create msgapp-helm Repository

```bash
# On GitHub, create a new repository: msgapp-helm

# Clone it locally
cd /home/gunda/Documents
git clone https://github.com/YOUR_USERNAME/msgapp-helm.git
cd msgapp-helm

# Copy Helm chart from msgapp repo
cp -r ../msgapp/helm-chart .

# Initialize and push
git add .
git commit -m "Initial Helm chart with Redis, RabbitMQ, and Worker"
git push origin main
```

## Step 2: Configure GitHub Secrets

### In msgapp repository (application code):

Go to: `Settings` → `Secrets and variables` → `Actions` → `New repository secret`

Add these secrets:

1. **ACR_USERNAME**
   ```bash
   az acr credential show --name your-acr-name --query username -o tsv
   ```

2. **ACR_PASSWORD**
   ```bash
   az acr credential show --name your-acr-name --query "passwords[0].value" -o tsv
   ```

3. **GH_PAT** (GitHub Personal Access Token)
   - Go to: https://github.com/settings/tokens
   - Generate new token (classic)
   - Select scope: `repo`
   - Copy and save as secret

### Update CI/CD Workflow

Edit `.github/workflows/ci-cd.yml`:

```yaml
env:
  ACR_NAME: your-acr-name
  ACR_LOGIN_SERVER: your-acr-name.azurecr.io
  HELM_REPO: YOUR_USERNAME/msgapp-helm
```

## Step 3: Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login via CLI
argocd login localhost:8080 --username admin --password <password-from-above>
```

Access ArgoCD UI: https://localhost:8080

## Step 4: Deploy Applications with ArgoCD

### Development Environment

```bash
# Update repository URL in argocd/application-dev.yaml
sed -i 's|YOUR_USERNAME|your-github-username|g' helm-chart/argocd/application-dev.yaml

# Apply ArgoCD application
kubectl apply -f helm-chart/argocd/application-dev.yaml

# Watch deployment
kubectl get pods -n msgapp-dev -w
```

### Staging Environment

```bash
# Update repository URL
sed -i 's|YOUR_USERNAME|your-github-username|g' helm-chart/argocd/application-staging.yaml

# Apply
kubectl apply -f helm-chart/argocd/application-staging.yaml

# Watch
kubectl get pods -n msgapp-staging -w
```

### Production Environment

```bash
# Update repository URL
sed -i 's|YOUR_USERNAME|your-github-username|g' helm-chart/argocd/application-prod.yaml

# Update production secrets in values-prod.yaml
# IMPORTANT: Replace placeholder passwords!

# Apply (manual sync for production)
kubectl apply -f helm-chart/argocd/application-prod.yaml

# Sync manually via ArgoCD UI or CLI
argocd app sync msgapp-prod
```

## Step 5: Test the Complete Workflow

### 1. Make a code change

```bash
cd /home/gunda/Documents/msgapp

# Make a change to your code
echo "// Test change" >> backend/backend/server.js

# Commit and push
git add .
git commit -m "Test CI/CD pipeline"
git push origin main
```

### 2. Watch GitHub Actions

- Go to: https://github.com/YOUR_USERNAME/msgapp/actions
- Watch the workflow build images
- Verify images are pushed to ACR
- Verify Helm repo is updated

### 3. Watch ArgoCD Deploy

```bash
# Watch ArgoCD detect changes
argocd app get msgapp-dev

# Watch pods rolling update
kubectl get pods -n msgapp-dev -w
```

### 4. Verify Deployment

```bash
# Check all components
kubectl get all -n msgapp-dev

# Check Redis
kubectl exec -it -n msgapp-dev deployment/msgapp-redis -- redis-cli ping

# Check RabbitMQ
kubectl port-forward -n msgapp-dev svc/msgapp-rabbitmq 15672:15672
# Open: http://localhost:15672 (user: msgapp, pass: from values.yaml)

# Check MongoDB
kubectl exec -it -n msgapp-dev msgapp-mongodb-0 -- mongosh --eval "db.adminCommand('ping')"

# Check application
kubectl port-forward -n msgapp-dev svc/msgapp-frontend 3000:3000
# Open: http://localhost:3000
```

## Step 6: Monitor and Manage

### ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open: https://localhost:8080

### View Application Status

```bash
# List all applications
argocd app list

# Get detailed status
argocd app get msgapp-dev

# View sync history
argocd app history msgapp-dev

# View logs
argocd app logs msgapp-dev
```

### Manual Sync (if needed)

```bash
# Sync application
argocd app sync msgapp-dev

# Sync specific resource
argocd app sync msgapp-dev --resource apps:Deployment:msgapp-backend

# Hard refresh (ignore cache)
argocd app sync msgapp-dev --force
```

### Rollback

```bash
# View history
argocd app history msgapp-dev

# Rollback to previous version
argocd app rollback msgapp-dev <REVISION_NUMBER>
```

## Troubleshooting

### Images Not Pulling from ACR

```bash
# Verify ACR credentials
az acr login --name your-acr-name

# Create image pull secret (if needed)
kubectl create secret docker-registry acr-secret \
  --docker-server=your-acr-name.azurecr.io \
  --docker-username=<username> \
  --docker-password=<password> \
  -n msgapp-dev

# Update values.yaml
imagePullSecrets:
  - name: acr-secret
```

### ArgoCD Out of Sync

```bash
# Check differences
argocd app diff msgapp-dev

# Force sync
argocd app sync msgapp-dev --force --prune
```

### Worker Not Processing Messages

```bash
# Check worker logs
kubectl logs -n msgapp-dev -l component=worker --tail=100 -f

# Check RabbitMQ queue
kubectl exec -it -n msgapp-dev deployment/msgapp-rabbitmq -- rabbitmqctl list_queues

# Check Redis connection
kubectl exec -it -n msgapp-dev deployment/msgapp-backend -- env | grep REDIS
```

### MongoDB Connection Issues

```bash
# Check MongoDB logs
kubectl logs -n msgapp-dev msgapp-mongodb-0 --tail=100

# Test connection from backend
kubectl exec -it -n msgapp-dev deployment/msgapp-backend -- \
  node -e "require('mongoose').connect(process.env.MONGO_URL).then(() => console.log('Connected')).catch(console.error)"
```

## Production Checklist

Before deploying to production:

- [ ] Update `values-prod.yaml` with secure passwords
- [ ] Configure TLS certificates (cert-manager)
- [ ] Set up monitoring (Prometheus/Grafana)
- [ ] Configure backup for MongoDB
- [ ] Enable HPA for all components
- [ ] Set resource limits appropriately
- [ ] Configure network policies
- [ ] Set up log aggregation
- [ ] Test disaster recovery procedures
- [ ] Document runbooks

## Next Steps

1. Set up monitoring with Prometheus/Grafana
2. Configure alerts for critical metrics
3. Implement backup strategy for MongoDB
4. Set up log aggregation (ELK/Loki)
5. Configure horizontal pod autoscaling
6. Implement network policies
7. Set up staging environment for testing

## Useful Commands

```bash
# View all resources in namespace
kubectl get all -n msgapp-dev

# Describe ArgoCD application
kubectl describe application msgapp-dev -n argocd

# Restart deployment
kubectl rollout restart deployment/msgapp-backend -n msgapp-dev

# Scale manually (if HPA disabled)
kubectl scale deployment/msgapp-backend --replicas=5 -n msgapp-dev

# View events
kubectl get events -n msgapp-dev --sort-by='.lastTimestamp'

# Delete and recreate application
kubectl delete -f helm-chart/argocd/application-dev.yaml
kubectl apply -f helm-chart/argocd/application-dev.yaml
```

## Architecture Diagram

```
Developer Push → GitHub → GitHub Actions
                              ↓
                         Build Images
                              ↓
                    Push to Azure ACR
                              ↓
                   Update msgapp-helm repo
                              ↓
                         ArgoCD Detects
                              ↓
                    Sync to Kubernetes
                              ↓
        ┌────────────────────┴────────────────────┐
        │         Kubernetes Cluster               │
        │  ┌──────────┐  ┌──────────┐            │
        │  │ Frontend │  │ Backend  │            │
        │  └────┬─────┘  └────┬─────┘            │
        │       │             │                    │
        │       │        ┌────┴─────┐             │
        │       │        │  Redis   │             │
        │       │        └────┬─────┘             │
        │       │             │                    │
        │       │        ┌────┴─────┐             │
        │       │        │ RabbitMQ │             │
        │       │        └────┬─────┘             │
        │       │             │                    │
        │       │        ┌────┴─────┐             │
        │       │        │  Worker  │             │
        │       │        └────┬─────┘             │
        │       │             │                    │
        │       └────────┬────┴─────┐             │
        │                │ MongoDB  │             │
        │                └──────────┘             │
        └──────────────────────────────────────────┘
```

Your GitOps workflow is now complete! 🚀
