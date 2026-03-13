# Envoy Gateway Configuration for MuleSoft Runtime Fabric (RTF)

## 1. Overview

This document outlines the steps to configure **Envoy Gateway** (using the Kubernetes Gateway API) as the ingress controller for a MuleSoft Runtime Fabric (RTF) cluster.

### Key Requirements

* **Protocol Support:** Explicitly separate endpoints for HTTPS (443) and HTTP (80).
* **Domain Handling:** Listens on the apex domain (`envoygateway.techworld360.io`) and wildcard subdomains (`*.envoygateway.techworld360.io`).
* **TLS Termination:** Managed at the Gateway level.
* **Path Rewriting:** Automatically strips the application base path for the **apex domain only**, routing traffic to the root (`/`) of the Mule app pod.

---

## 2. Prerequisites

* Running Kubernetes cluster with MuleSoft RTF installed.
* `kubectl` CLI with cluster admin access.
* `helm` CLI installed.
* TLS certificate (`fullchain.crt`) and private key (`private.key`) for the domains.

---

## 3. Installation & Setup

### Step 3.1: Install Envoy Gateway

Install the Envoy Gateway controller using Helm to provision the Gateway API CRDs and control plane.

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.0 \
  -n envoy-gateway-system \
  --create-namespace

# Wait for deployment availability
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

```

### Step 3.2: Create the GatewayClass

Link your Gateway resources to the Envoy controller.

```yaml
# gateway-class.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller

```

**Apply:** `kubectl apply -f gateway-class.yaml`

### Step 3.3: Configure TLS Certificate Secret

Store certificates in the `envoy-gateway-system` namespace.

```bash
kubectl create secret tls envoy-tls-secret \
  --namespace envoy-gateway-system \
  --cert=path/to/fullchain.crt \
  --key=path/to/private.key

```

### Step 3.4: Deploy the Gateway

The following configuration defines listeners for ports 80 and 443 across both apex and wildcard hostnames.

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: rtf-envoy-gateway
  namespace: envoy-gateway-system
spec:
  gatewayClassName: envoy-gateway
  listeners:
    # --- HTTPS Listeners (Port 443) ---
    - name: https-wildcard
      protocol: HTTPS
      port: 443
      hostname: "*.envoygateway.techworld360.io"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - name: envoy-tls-secret
            kind: Secret

    - name: https-root
      protocol: HTTPS
      port: 443
      hostname: "envoygateway.techworld360.io"
      allowedRoutes:
        namespaces:
          from: All
      tls:
        mode: Terminate
        certificateRefs:
          - name: envoy-tls-secret
            kind: Secret

    # --- HTTP Listeners (Port 80) ---
    - name: http-wildcard
      protocol: HTTP
      port: 80
      hostname: "*.envoygateway.techworld360.io"
      allowedRoutes:
        namespaces:
          from: All

    - name: http-root
      protocol: HTTP
      port: 80
      hostname: "envoygateway.techworld360.io"
      allowedRoutes:
        namespaces:
          from: All

```

**Apply:** `kubectl apply -f gateway.yaml`

---

## 4. MuleSoft RTF Configuration (HTTPRouteTemplates)

RTF uses `HTTPRouteTemplate` CRDs to generate `HTTPRoute` manifests during deployment. We deploy four templates to handle the specific logic for each domain/protocol combination.

### Template 1: HTTPS Apex Domain (With Path Rewrite)

Uses `ReplacePrefixMatch` to strip `{{ .Path }}`.

```yaml
# rtf-template-https-root.yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: envoy-template-https-root-rewrite
  namespace: rtf
spec:
  baseEndpoints:
    - https://envoygateway.techworld360.io
  resources:
    - |
      apiVersion: gateway.networking.k8s.io/v1
      kind: HTTPRoute
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        parentRefs:
          - kind: Gateway
            name: rtf-envoy-gateway
            namespace: envoy-gateway-system
        hostnames:
          - {{ .Host }}
        rules:
          - matches:
              - path:
                  type: PathPrefix
                  value: {{ .Path }}
            filters:
              - type: URLRewrite
                urlRewrite:
                  path:
                    type: ReplacePrefixMatch
                    replacePrefixMatch: /
            backendRefs:
              - name: {{ .Service.Name }}
                port: {{ .Service.Port }}

```

### Template 2: HTTP Apex Domain (With Path Rewrite)

```yaml
# rtf-template-http-root.yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: envoy-template-http-root-rewrite
  namespace: rtf
spec:
  baseEndpoints:
    - http://envoygateway.techworld360.io
  resources:
    - |
      apiVersion: gateway.networking.k8s.io/v1
      kind: HTTPRoute
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        parentRefs:
          - kind: Gateway
            name: rtf-envoy-gateway
            namespace: envoy-gateway-system
        hostnames:
          - {{ .Host }}
        rules:
          - matches:
              - path:
                  type: PathPrefix
                  value: {{ .Path }}
            filters:
              - type: URLRewrite
                urlRewrite:
                  path:
                    type: ReplacePrefixMatch
                    replacePrefixMatch: /
            backendRefs:
              - name: {{ .Service.Name }}
                port: {{ .Service.Port }}

```

### Template 3: HTTPS Wildcard Domain (No Rewrite)

```yaml
# rtf-template-https-wildcard.yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: envoy-template-https-wildcard-norewrite
  namespace: rtf
spec:
  baseEndpoints:
    - https://*.envoygateway.techworld360.io
  resources:
    - |
      apiVersion: gateway.networking.k8s.io/v1
      kind: HTTPRoute
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        parentRefs:
          - kind: Gateway
            name: rtf-envoy-gateway
            namespace: envoy-gateway-system
        hostnames:
          - {{ .Host }}
        rules:
          - matches:
              - path:
                  type: PathPrefix
                  value: {{ .Path }}
            backendRefs:
              - name: {{ .Service.Name }}
                port: {{ .Service.Port }}

```

### Template 4: HTTP Wildcard Domain (No Rewrite)

```yaml
# rtf-template-http-wildcard.yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: envoy-template-http-wildcard-norewrite
  namespace: rtf
spec:
  baseEndpoints:
    - http://*.envoygateway.techworld360.io
  resources:
    - |
      apiVersion: gateway.networking.k8s.io/v1
      kind: HTTPRoute
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        parentRefs:
          - kind: Gateway
            name: rtf-envoy-gateway
            namespace: envoy-gateway-system
        hostnames:
          - {{ .Host }}
        rules:
          - matches:
              - path:
                  type: PathPrefix
                  value: {{ .Path }}
            backendRefs:
              - name: {{ .Service.Name }}
                port: {{ .Service.Port }}

```

---

## 5. Grant RTF Permissions

Apply the following RBAC to allow the RTF Agent to manage the `HTTPRoute` resources.

```yaml
# rtf-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rtf-manage-gateway-httproutes
rules:
  - apiGroups:
      - gateway.networking.k8s.io
    resources:
      - httproutes
    verbs:
      - '*'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rtf-manage-gateway-httproutes-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: rtf-manage-gateway-httproutes
subjects:
  - kind: ServiceAccount
    name: rtf-agent
    namespace: rtf

```