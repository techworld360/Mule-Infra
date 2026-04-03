# Traefik Gateway API: Traffic Control & Security Cheat Sheet

This document covers the specific implementation of URL rewrites, timeouts, CORS, and IP whitelisting for Kubernetes Gateway API using Traefik.

---

## 1. URL Rewriting (Path Manipulation)
The `URLRewrite` filter modifies the HTTP request path before it reaches the backend pod. 

### A. Prefix Stripping (Standard for RTF)
Removes the base path (e.g., `/my-app`) and sends only the remaining path to the pod.
```yaml
filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /
```

### B. Full Path Override
Forces every request to a specific internal path, regardless of what the user typed.
```yaml
filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplaceFullPath
        replaceFullPath: /health-check
```

### C. Advanced "Multiple" Rewrites (Regex)
Standard Gateway API only allows **one** `URLRewrite` filter per rule. For complex or multiple logic steps (e.g., changing `/api/v1/users/5` to `/users?id=5`), you must use a Traefik **Middleware**.

**1. Create the Middleware:**
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: advanced-regex-rewrite
  namespace: rtf
spec:
  replacePathRegex:
    regex: "^/api/v1/([^/]+)/([^/]+)"
    replacement: "/$1/$2?version=v1"
```

**2. Reference in HTTPRoute:**
```yaml
filters:
  - type: ExtensionRef
    extensionRef:
      group: traefik.io
      kind: Middleware
      name: advanced-regex-rewrite
```

---

## 2. Timeouts & Keep-Alive
Timeouts are split into "Client-to-Gateway" and "Gateway-to-Backend."

### A. Gateway-to-Backend (Route Level)
Defined inside the `HTTPRoute` to control how long Traefik waits for the Mule pod.
```yaml
rules:
  - timeouts:
      request: 300s        # Total time for the full request/response
      backendRequest: 60s  # Time spent waiting for the pod to start responding
    matches:
      - path: { type: PathPrefix, value: /api }
    backendRefs:
      - name: my-mule-service
        port: 8081
```

### B. Client-to-Gateway (Global Level)
Configured in your Helm `values.yaml` for the physical ports.
```yaml
ports:
  websecure:
    transport:
      respondingTimeouts:
        readTimeout: 60s
        writeTimeout: 60s
        idleTimeout: 180s # Connection Keep-Alive TTL
```

---

## 3. Security Middlewares (CORS & IP Whitelist)



### A. CORS Policy
Allows cross-domain requests from specific front-ends.
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rtf-cors-policy
  namespace: rtf
spec:
  headers:
    accessControlAllowMethods: ["GET", "POST", "OPTIONS"]
    accessControlAllowOriginList: ["https://portal.example.com"]
    accessControlMaxAge: 86400
```

### B. IP Whitelisting
Restricts access to specific subnets or internal VPN IPs.
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: rtf-ip-whitelist
  namespace: rtf
spec:
  ipWhiteList:
    sourceRange:
      - "10.0.0.0/8"    # Internal Network
      - "192.168.1.50"  # Dedicated Admin IP
```

---

## 4. Implementation: Stacking Filters
You can apply multiple security policies and a rewrite to a single route by listing them in order.

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: traefik-advanced-template
  namespace: rtf
spec:
  baseEndpoints:
    - https://*.your-mule-rtf.local
  resources:
    - |
      apiVersion: gateway.networking.k8s.io/v1
      kind: HTTPRoute
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        parentRefs:
          - name: traefik-gateway
            namespace: traefik
        hostnames:
          - {{ .Host }}
        rules:
          # --- SECTION: TIMEOUTS ---
          - timeouts:
              request: 300s        # Total lifecycle (matches Nginx proxy-body-timeout)
              backendRequest: 60s   # Connection to Mule Pod (matches Nginx proxy-read-timeout)
            
            matches:
              - path:
                  type: PathPrefix
                  value: {{ .Path }}
            
            # --- SECTION: FILTERS (CORS, IP, REWRITES) ---
            filters:
              # 1. Apply Security (IP Whitelist & CORS)
              - type: ExtensionRef
                extensionRef:
                  group: traefik.io
                  kind: Middleware
                  name: rtf-security-stack
              
              # 2. URL Rewrite: Standard Prefix Strip
              # Converts /app-name/api/v1 -> /api/v1
              - type: URLRewrite
                urlRewrite:
                  path:
                    type: ReplacePrefixMatch
                    replacePrefixMatch: /

            backendRefs:
              - name: {{ .Service.Name }}
                port: {{ .Service.Port }}
```
## 5. Handling Multiple/Complex Rewrites

If your requirement for "multiple rewrites" involves complex logic (like changing versions or reordering path segments) that a standard `ReplacePrefixMatch` cannot handle, you must replace the `URLRewrite` filter with a **Traefik Regex Middleware**.

### Step A: Define the Regex Middleware
```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: complex-path-rewrite
  namespace: rtf
spec:
  replacePathRegex:
    # Example: Extracts 'version' and 'resource' to reorder them
    regex: "^/([^/]+)/api/(v[0-9])/(.*)"
    replacement: "/api/$2/$3?originalApp=$1"
```

### Step B: Update the Template Filter
Inside the `HTTPRouteTemplate`, you would remove the `type: URLRewrite` block and add:
```yaml
              - type: ExtensionRef
                extensionRef:
                  group: traefik.io
                  kind: Middleware
                  name: complex-path-rewrite
```

---

## 6. Summary of Configuration logic

| Feature | Implementation Method | Location |
| :--- | :--- | :--- |
| **Simple Rewrite** | `URLRewrite` filter | Inline in `HTTPRouteTemplate` |
| **Complex/Regex Rewrite** | `replacePathRegex` | Middleware + `ExtensionRef` |
| **Request Timeouts** | `timeouts` block | Inline in `HTTPRouteTemplate` |
| **CORS** | `headers` | Middleware + `ExtensionRef` |
| **IP Whitelisting** | `ipWhiteList` | Middleware + `ExtensionRef` |
| **Keep-Alive (TTL)** | `transport.respondingTimeouts` | Helm `values.yaml` (Global) |

---
