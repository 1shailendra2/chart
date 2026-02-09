# Ingress URL Rewrite Fix

## The Problem You Caught

Your backend routes are defined as:
```javascript
app.use('/api/auth', authRoutes);
app.use('/api/messages', messageRoutes);
app.use('/api/rooms', roomRoutes);
```

The backend **expects the full path** including `/api`.

## What Was Wrong

The original Ingress had:
```yaml
nginx.ingress.kubernetes.io/rewrite-target: /$2
path: /api(/|$)(.*)
```

This was **rewriting URLs**:
```
Browser sends:    /api/auth/login
Ingress rewrites: /auth/login  ❌
Backend receives: /auth/login  (NO MATCH!)
```

## The Fix

Removed the rewrite and simplified to prefix matching:
```yaml
# Removed: nginx.ingress.kubernetes.io/rewrite-target: /$2
path: /api
pathType: Prefix
```

Now the **full path passes through**:
```
Browser sends:    /api/auth/login
Ingress passes:   /api/auth/login  ✅
Backend receives: /api/auth/login  (MATCHES!)
```

## Updated Routing

### Backend API
```yaml
- path: /api
  pathType: Prefix
```
Matches: `/api/auth/login`, `/api/messages`, `/api/rooms`

### Socket.io
```yaml
- path: /socket.io
  pathType: Prefix
```
Matches: `/socket.io/?transport=websocket`

### Frontend
```yaml
- path: /
  pathType: Prefix
```
Matches: Everything else

## Request Flow (Fixed)

```
User → https://msgapp.local/api/auth/login
     → Ingress matches /api (prefix)
     → Routes to msgapp-backend:5000
     → Backend receives /api/auth/login
     → Matches app.use('/api/auth', authRoutes) ✅
     → authRoutes handles /login
     → Success!
```

Great catch! The routes now work correctly. 🎯
