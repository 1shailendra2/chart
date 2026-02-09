# Sticky Session Configuration for Socket.io

## Overview

Your Helm Chart now includes **sticky session support** to ensure Socket.io connections remain connected to the same backend pod throughout the session lifecycle.

## Configuration Details

### Backend Service - Layer 4 Sticky Sessions

**File**: `helm-chart/templates/backend-service.yaml`

```yaml
spec:
  type: ClusterIP
  # Enable sticky sessions for Socket.io connections
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 hours
```

**How it works**:
- Routes requests from the same client IP to the same backend pod
- Session timeout: **3 hours** (10800 seconds)
- Works at the Kubernetes Service level (Layer 4)

### Ingress - Layer 7 Sticky Sessions

**File**: `helm-chart/templates/ingress.yaml`

```yaml
annotations:
  # Enable sticky sessions for Socket.io connections
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/affinity-mode: "persistent"
  nginx.ingress.kubernetes.io/session-cookie-name: "msgapp-sticky"
  nginx.ingress.kubernetes.io/session-cookie-max-age: "10800"  # 3 hours
  nginx.ingress.kubernetes.io/session-cookie-path: "/"
```

**How it works**:
- Uses HTTP cookies to maintain session affinity
- Cookie name: `msgapp-sticky`
- Cookie lifetime: **3 hours** (10800 seconds)
- Works at the Ingress level (Layer 7)
- More reliable than IP-based affinity (handles proxy/NAT scenarios)

## Why Both Layers?

**Defense in depth approach**:

1. **Ingress (L7)**: Primary sticky session mechanism
   - Cookie-based affinity works even when clients are behind NAT/proxies
   - Survives client IP changes (mobile networks, VPNs)
   
2. **Service (L4)**: Backup mechanism
   - Works for direct service-to-service communication
   - Provides affinity even if cookies are disabled/cleared

## MongoDB - No Sticky Sessions Needed

**File**: `helm-chart/templates/mongodb-statefulset.yaml`

MongoDB uses a **StatefulSet** with:
- Single replica (no load balancing needed)
- Stable network identity: `msgapp-mongodb-0`
- Headless service (direct pod addressing)

**Result**: All backend pods always connect to the same MongoDB instance.

## Testing Sticky Sessions

### 1. Verify Cookie is Set

```bash
curl -v http://msgapp.local/api/health
# Look for: Set-Cookie: msgapp-sticky=...
```

### 2. Test Socket.io Connection Persistence

```bash
# Connect to the application and check backend logs
kubectl logs -f -l app.kubernetes.io/component=backend

# You should see the same pod handling all requests from a single client
```

### 3. Verify Service Session Affinity

```bash
kubectl describe svc msgapp-backend | grep -A 5 "Session Affinity"
# Should show: ClientIP with timeout 10800
```

## Configuration Options

You can customize the session timeout in `values.yaml`:

```yaml
backend:
  service:
    sessionAffinityTimeout: 10800  # seconds (3 hours)

ingress:
  sessionCookieMaxAge: 10800  # seconds (3 hours)
```

## Important Notes

⚠️ **Session Timeout**: 3 hours is suitable for chat applications. Adjust based on your use case:
- Shorter (1800s = 30 min): Better for high-traffic scenarios
- Longer (21600s = 6 hours): Better for long-running sessions

⚠️ **Pod Scaling**: When scaling down backend pods, some active sessions will be terminated. Consider:
- Graceful shutdown periods
- Connection retry logic in the frontend
- Horizontal Pod Autoscaler (HPA) with careful min/max settings

⚠️ **Cookie Security**: For production, add these annotations:

```yaml
nginx.ingress.kubernetes.io/session-cookie-secure: "true"
nginx.ingress.kubernetes.io/session-cookie-samesite: "Strict"
```

## Summary

✅ **Backend Service**: ClientIP session affinity (3 hours)  
✅ **Ingress**: Cookie-based sticky sessions (3 hours)  
✅ **MongoDB**: Single StatefulSet pod (no affinity needed)  
✅ **Socket.io**: Connections persist to the same backend pod  

Your Socket.io connections are now properly configured for multi-pod deployments!
