# Horizontal Pod Autoscaler (HPA) Configuration

## Overview

The Helm Chart now includes **HPA (Horizontal Pod Autoscaler)** for both frontend and backend to automatically scale pods based on CPU and memory usage.

## HPA Templates

### Backend HPA
**File**: `helm-chart/templates/backend-hpa.yaml`

Automatically scales `msgapp-backend` deployment based on resource utilization.

### Frontend HPA
**File**: `helm-chart/templates/frontend-hpa.yaml`

Automatically scales `msgapp-frontend` deployment based on resource utilization.

## Configuration (values.yaml)

### Backend Autoscaling

```yaml
backend:
  autoscaling:
    enabled: false              # Set to true to enable HPA
    minReplicas: 2              # Minimum number of pods
    maxReplicas: 10             # Maximum number of pods
    targetCPUUtilizationPercentage: 80     # Scale up when CPU > 80%
    targetMemoryUtilizationPercentage: 80  # Scale up when Memory > 80%
```

### Frontend Autoscaling

```yaml
frontend:
  autoscaling:
    enabled: false              # Set to true to enable HPA
    minReplicas: 2              # Minimum number of pods
    maxReplicas: 10             # Maximum number of pods
    targetCPUUtilizationPercentage: 80     # Scale up when CPU > 80%
    targetMemoryUtilizationPercentage: 80  # Scale up when Memory > 80%
```

## How It Works

### Scaling Triggers

HPA monitors both CPU and memory metrics:
- **CPU**: Scales when average CPU usage exceeds 80%
- **Memory**: Scales when average memory usage exceeds 80%

### Scale-Up Behavior

**Fast and aggressive**:
- No stabilization window (immediate scale-up)
- Can scale up by 100% or add 2 pods, whichever is greater
- Checks every 30 seconds

Example: If you have 2 pods and CPU hits 90%, HPA can immediately add 2 more pods.

### Scale-Down Behavior

**Slow and cautious**:
- 5-minute stabilization window (prevents flapping)
- Can only scale down by 50% at a time
- Checks every 60 seconds

Example: If you have 10 pods and load decreases, HPA will wait 5 minutes, then remove at most 5 pods.

## Enabling HPA

### Option 1: Update values.yaml

```yaml
backend:
  autoscaling:
    enabled: true  # Enable HPA
```

### Option 2: Command Line

```bash
helm install msgapp ./helm-chart \
  --set backend.autoscaling.enabled=true \
  --set frontend.autoscaling.enabled=true
```

### Option 3: Custom Values File

```yaml
# production-values.yaml
backend:
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70

frontend:
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 15
    targetCPUUtilizationPercentage: 75
```

```bash
helm install msgapp ./helm-chart -f production-values.yaml
```

## Important Notes

### ⚠️ HPA vs Manual Replicas

When HPA is **enabled**:
- `replicaCount` in values.yaml is **ignored**
- HPA controls the number of replicas
- Use `minReplicas` and `maxReplicas` instead

When HPA is **disabled**:
- `replicaCount` in values.yaml is used
- Manual scaling only

### ⚠️ Resource Requests Required

HPA **requires** resource requests to be defined:

```yaml
backend:
  resources:
    requests:
      cpu: 250m      # REQUIRED for CPU-based autoscaling
      memory: 256Mi  # REQUIRED for memory-based autoscaling
```

Already configured in your `values.yaml` ✅

### ⚠️ Metrics Server Required

Your Kubernetes cluster must have **metrics-server** installed:

```bash
# Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system

# If not installed, install it
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### ⚠️ Sticky Sessions Consideration

With HPA enabled and Socket.io sticky sessions:
- New pods are added seamlessly (new connections go to new pods)
- When scaling down, active connections on terminated pods will disconnect
- Ensure your frontend has reconnection logic

## Monitoring HPA

### Check HPA Status

```bash
# View HPA status
kubectl get hpa

# Detailed HPA info
kubectl describe hpa msgapp-backend-hpa
kubectl describe hpa msgapp-frontend-hpa

# Watch HPA in real-time
kubectl get hpa -w
```

### Example Output

```
NAME                  REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS
msgapp-backend-hpa    Deployment/msgapp-backend   45%/80%, 60%/80%   2         10        3
msgapp-frontend-hpa   Deployment/msgapp-frontend  30%/80%, 40%/80%   2         10        2
```

This shows:
- Backend: CPU at 45%, Memory at 60% → 3 replicas running
- Frontend: CPU at 30%, Memory at 40% → 2 replicas running (at minimum)

## Testing HPA

### Generate Load

```bash
# Port-forward to backend
kubectl port-forward svc/msgapp-backend 5000:5000

# Generate CPU load (run in multiple terminals)
while true; do curl http://localhost:5000/api/rooms; done
```

### Watch Scaling

```bash
# Terminal 1: Watch HPA
kubectl get hpa -w

# Terminal 2: Watch pods
kubectl get pods -l app=msgapp,component=backend -w
```

You should see:
1. CPU/Memory metrics increase
2. HPA triggers scale-up
3. New pods are created
4. Load distributes across pods
5. Metrics stabilize

## Production Recommendations

### High Traffic Application

```yaml
backend:
  autoscaling:
    enabled: true
    minReplicas: 5      # Always have 5 pods minimum
    maxReplicas: 50     # Can scale up to 50 pods
    targetCPUUtilizationPercentage: 70  # Scale earlier at 70%
```

### Cost-Optimized

```yaml
backend:
  autoscaling:
    enabled: true
    minReplicas: 1      # Start with just 1 pod
    maxReplicas: 5      # Limit to 5 pods max
    targetCPUUtilizationPercentage: 85  # Scale only when really needed
```

### Balanced (Recommended)

```yaml
backend:
  autoscaling:
    enabled: true
    minReplicas: 2      # Redundancy with 2 pods
    maxReplicas: 10     # Room to grow
    targetCPUUtilizationPercentage: 80  # Standard threshold
```

## Summary

✅ **Backend HPA**: `backend-hpa.yaml` created  
✅ **Frontend HPA**: `frontend-hpa.yaml` created  
✅ **Configuration**: Added to `values.yaml`  
✅ **Disabled by default**: Set `enabled: true` to activate  
✅ **Smart scaling**: Fast scale-up, slow scale-down  
✅ **Dual metrics**: CPU and memory-based autoscaling  

Enable HPA for production deployments to handle traffic spikes automatically! 🚀
