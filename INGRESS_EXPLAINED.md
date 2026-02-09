# Deep Dive: Ingress.yaml Explained

## Complete File Breakdown

Let me explain **every single line** of your Ingress configuration.

---

## Line 1: Conditional Rendering

```yaml
{{- if .Values.ingress.enabled }}
```

**What it does**: Only creates the Ingress if `ingress.enabled: true` in values.yaml

**Helm template syntax**:
- `{{- ... }}` = Helm template directive
- `-` = Trim whitespace
- `.Values.ingress.enabled` = Read from values.yaml

**Why**: Allows you to disable Ingress for local development or when using different routing methods.

---

## Lines 2-3: Kubernetes Resource Definition

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
```

**apiVersion**: `networking.k8s.io/v1`
- Kubernetes API group for networking resources
- `v1` = Stable API version (introduced in Kubernetes 1.19)
- Older versions used `networking.k8s.io/v1beta1`

**kind**: `Ingress`
- Defines this as an Ingress resource
- Ingress = HTTP/HTTPS routing rules to services

---

## Lines 4-8: Metadata

```yaml
metadata:
  name: msgapp-ingress
  labels:
    app: msgapp
    component: ingress
```

**name**: `msgapp-ingress`
- Unique identifier for this Ingress
- Used in `kubectl get ingress msgapp-ingress`

**labels**: Key-value pairs for organization
- `app: msgapp` = Groups all msgapp resources
- `component: ingress` = Identifies this as the ingress component
- Used for filtering: `kubectl get ingress -l app=msgapp`

---

## Lines 9-20: Annotations (NGINX Configuration)

Annotations are **NGINX-specific configurations** that control how the Ingress Controller behaves.

### Sticky Session Annotations (Lines 11-15)

```yaml
nginx.ingress.kubernetes.io/affinity: "cookie"
```
**What**: Enable session affinity using cookies
**Why**: Ensures Socket.io connections always go to the same backend pod
**How**: NGINX sets a cookie on first request, uses it for routing subsequent requests

```yaml
nginx.ingress.kubernetes.io/affinity-mode: "persistent"
```
**What**: Cookie persists across browser sessions
**Options**:
- `persistent` = Cookie survives browser restart
- `balanced` = Cookie expires when browser closes

```yaml
nginx.ingress.kubernetes.io/session-cookie-name: "msgapp-sticky"
```
**What**: Name of the sticky session cookie
**Result**: Browser receives `Set-Cookie: msgapp-sticky=<hash>`
**Why custom name**: Easier to debug, avoids conflicts with other apps

```yaml
nginx.ingress.kubernetes.io/session-cookie-max-age: "10800"
```
**What**: Cookie lifetime in seconds
**Value**: `10800` = 3 hours
**Why**: Long enough for chat sessions, short enough to rebalance load

```yaml
nginx.ingress.kubernetes.io/session-cookie-path: "/"
```
**What**: Cookie is valid for all paths
**Why**: Socket.io and API calls both need sticky sessions

### URL Rewrite Annotation (Line 17)

```yaml
nginx.ingress.kubernetes.io/rewrite-target: /$2
```

**What**: Rewrites the URL before sending to backend
**How it works**:

**Example 1 - API Request**:
```
Browser sends:     GET /api/auth/login
Path matches:      /api(/|$)(.*)
Capture groups:    $1 = /    $2 = auth/login
Rewrite to:        /auth/login
Backend receives:  GET /auth/login
```

**Example 2 - Socket.io**:
```
Browser sends:     GET /socket.io/?transport=polling
Path matches:      /socket.io(/|$)(.*)
Capture groups:    $1 = /    $2 = ?transport=polling
Rewrite to:        /?transport=polling
Backend receives:  GET /?transport=polling
```

**Why**: Your backend expects `/auth/login`, not `/api/auth/login`

### Custom Annotations (Lines 18-20)

```yaml
{{- if .Values.ingress.annotations }}
{{- toYaml .Values.ingress.annotations | nindent 4 }}
{{- end }}
```

**What**: Allows adding custom annotations from values.yaml
**Example usage**:
```yaml
# values.yaml
ingress:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
```

---

## Lines 21-24: Ingress Class

```yaml
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
```

**What**: Specifies which Ingress Controller handles this Ingress
**Value**: `nginx` (from values.yaml)

**Why it matters**:
- Kubernetes can have multiple Ingress Controllers (nginx, traefik, HAProxy, etc.)
- `ingressClassName: nginx` = Only NGINX Ingress Controller processes this
- Without this, all controllers might try to handle it

---

## Lines 25-30: TLS Configuration

```yaml
{{- if .Values.ingress.tls.enabled }}
tls:
  - hosts:
      - {{ .Values.ingress.host }}
    secretName: {{ .Values.ingress.tls.secretName }}
{{- end }}
```

**What**: Enables HTTPS with TLS certificates

**When enabled** (values.yaml: `tls.enabled: true`):
```yaml
tls:
  - hosts:
      - msgapp.example.com
    secretName: msgapp-tls
```

**How it works**:
1. Certificate stored in Kubernetes secret `msgapp-tls`
2. NGINX terminates TLS at the Ingress
3. Backend communication is HTTP (inside cluster)

**Creating the TLS secret**:
```bash
kubectl create secret tls msgapp-tls \
  --cert=path/to/cert.crt \
  --key=path/to/cert.key
```

**Or with cert-manager** (automatic):
```yaml
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
```

---

## Lines 31-58: Routing Rules

### Host Rule (Line 32)

```yaml
rules:
  - host: {{ .Values.ingress.host }}
```

**What**: Only handle requests for this hostname
**Value**: `msgapp.local` (from values.yaml)
**Result**: Requests to `msgapp.local` are routed, others are rejected

**DNS requirement**: `msgapp.local` must resolve to your Ingress Controller's IP

---

### Path Rules (Lines 34-58)

**Order matters!** More specific paths must come first.

#### Rule 1: Backend API (Lines 36-42)

```yaml
- path: /api(/|$)(.*)
  pathType: ImplementationSpecific
  backend:
    service:
      name: msgapp-backend
      port:
        number: 5000
```

**path**: `/api(/|$)(.*)`
- Regex pattern with capture groups
- `/api` = Literal match
- `(/|$)` = Followed by `/` or end of string (captured as `$1`)
- `(.*)` = Capture everything after (captured as `$2`)

**Matches**:
- ✅ `/api/auth/login` → `$2 = auth/login`
- ✅ `/api/rooms` → `$2 = rooms`
- ✅ `/api` → `$2 = ` (empty)
- ❌ `/apiv2/test` → No match (missing `/` or end)

**pathType**: `ImplementationSpecific`
- Allows regex patterns (NGINX-specific)
- Other options: `Exact`, `Prefix`

**backend**: Routes to `msgapp-backend` service on port `5000`

**Full flow**:
```
User → https://msgapp.local/api/auth/login
     → Ingress matches /api(/|$)(.*)
     → Rewrites to /auth/login (via rewrite-target: /$2)
     → Routes to msgapp-backend:5000
     → Backend receives GET /auth/login
```

#### Rule 2: Socket.io (Lines 44-50)

```yaml
- path: /socket.io(/|$)(.*)
  pathType: ImplementationSpecific
  backend:
    service:
      name: msgapp-backend
      port:
        number: 5000
```

**Why separate rule**: Socket.io has specific path requirements

**Matches**:
- ✅ `/socket.io/?transport=polling`
- ✅ `/socket.io/?transport=websocket`
- ✅ `/socket.io/socket.io.js`

**Full flow**:
```
Browser → wss://msgapp.local/socket.io/?transport=websocket
        → Ingress matches /socket.io(/|$)(.*)
        → Rewrites to /?transport=websocket
        → Routes to msgapp-backend:5000
        → Backend Socket.io server handles it
```

#### Rule 3: Frontend Catch-All (Lines 52-58)

```yaml
- path: /
  pathType: Prefix
  backend:
    service:
      name: msgapp-frontend
      port:
        number: 3000
```

**path**: `/`
**pathType**: `Prefix`
- Matches any path starting with `/`
- This is a **catch-all** for everything not matched above

**Matches**:
- ✅ `/` → Homepage
- ✅ `/login` → Login page
- ✅ `/dashboard` → Dashboard
- ✅ `/static/css/main.css` → Static files

**Why last**: If this came first, it would match everything (including `/api`)

**Full flow**:
```
User → https://msgapp.local/
     → No match for /api or /socket.io
     → Matches / (catch-all)
     → Routes to msgapp-frontend:3000
     → Next.js serves the page
```

---

## Request Flow Examples

### Example 1: User Visits Homepage

```
1. Browser → https://msgapp.local/
2. DNS → Resolves to Ingress Controller IP
3. Ingress → Matches path: /
4. Routes to → msgapp-frontend:3000
5. Frontend → Returns Next.js HTML
6. Browser → Renders page
```

### Example 2: User Logs In

```
1. Browser → POST https://msgapp.local/api/auth/login
2. Ingress → Matches path: /api(/|$)(.*)
3. Captures → $2 = auth/login
4. Rewrites → /auth/login
5. Routes to → msgapp-backend:5000
6. Backend → Processes login, returns JWT
7. Ingress → Sets sticky cookie: msgapp-sticky=abc123
8. Browser → Receives response + cookie
```

### Example 3: Socket.io Connection

```
1. Browser → ws://msgapp.local/socket.io/?transport=websocket
2. Ingress → Matches path: /socket.io(/|$)(.*)
3. Reads cookie → msgapp-sticky=abc123
4. Routes to → Same backend pod (sticky session)
5. Backend → Upgrades to WebSocket
6. Connection → Established for real-time chat
```

---

## Common Annotations Reference

Here are other useful NGINX Ingress annotations you might need:

### Rate Limiting
```yaml
nginx.ingress.kubernetes.io/limit-rps: "10"  # 10 requests per second
```

### CORS
```yaml
nginx.ingress.kubernetes.io/enable-cors: "true"
nginx.ingress.kubernetes.io/cors-allow-origin: "*"
```

### Request Size
```yaml
nginx.ingress.kubernetes.io/proxy-body-size: "10m"  # Max 10MB uploads
```

### Timeouts
```yaml
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"  # 1 hour for WebSockets
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
```

### SSL Redirect
```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true"  # Force HTTPS
```

### Custom Error Pages
```yaml
nginx.ingress.kubernetes.io/custom-http-errors: "404,503"
nginx.ingress.kubernetes.io/default-backend: custom-error-pages
```

---

## Debugging Commands

```bash
# View Ingress details
kubectl describe ingress msgapp-ingress

# Check Ingress Controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Test DNS resolution
nslookup msgapp.local

# Get Ingress IP/hostname
kubectl get ingress msgapp-ingress

# Test from inside cluster
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- \
  curl http://msgapp-backend:5000/health
```

---

## Summary

Your Ingress is a **smart HTTP router** that:

1. ✅ **Routes traffic** based on URL paths
2. ✅ **Enables sticky sessions** for Socket.io
3. ✅ **Rewrites URLs** to match backend expectations
4. ✅ **Terminates TLS** (when enabled)
5. ✅ **Load balances** across backend pods
6. ✅ **Provides a single entry point** for your entire application

**Traffic flow**:
```
Internet → Ingress Controller → Ingress Rules → Services → Pods
```

This is the **gateway** to your entire application! 🚀
