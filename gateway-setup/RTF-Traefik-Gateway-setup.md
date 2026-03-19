
# Traefik Gateway API for MuleSoft RTF

This guide provides a structured, command-line-driven approach to deploying **Traefik Proxy** as the **Gateway API** ingress for **MuleSoft Runtime Fabric (RTF) 3.x**. 

This implementation strictly adheres to the Kubernetes Gateway API standard and utilizes Traefik Helm Chart **v39.0.1**.

---

## 📋 Prerequisites
* A running Kubernetes cluster with **MuleSoft RTF 3.x** installed.
* `kubectl` and `helm` installed and configured.
* `openssl` for certificate generation.

---

## 🏗️ Step 1: Install Gateway API CRDs & Traefik RBAC
The Gateway API is an extension and must be installed manually. You must also grant Traefik the necessary permissions to manage these resources.

```bash
# Install standard Gateway API CRDs (v1.2.1)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml

# Install Traefik RBAC for Gateway API
kubectl apply -f https://raw.githubusercontent.com/traefik/traefik/v3.1/docs/content/reference/dynamic-configuration/kubernetes-gateway-rbac.yml
```

---

## 🔐 Step 2: Prepare Namespace & TLS Certificate
Create the namespace and the TLS secret first to ensure the Gateway initializes in a healthy state.

```bash
# 1. Create the target namespace
kubectl create namespace traefik

# 2. Generate a self-signed certificate 
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 365 -nodes -subj "/CN=your-mule-rtf.local"

# 3. Store the certificate as a Kubernetes Secret
kubectl create secret tls traefik-default-cert \
  --cert=cert.pem \
  --key=key.pem \
  -n traefik
```

---

## 🚢 Step 3: Install Traefik via Helm
Create a `values.yaml` file to map Traefik's internal non-root ports to standard external ports and apply your custom certificate.

### `values.yaml`
```yaml
providers:
  kubernetesGateway:
    enabled: true
  kubernetesIngress:
    enabled: false 

api:
  dashboard: true
  insecure: true      

ports:
  traefik:
    port: 9000          
    expose:
      default: false    
  web:
    port: 8000          
    exposedPort: 80     
    expose:
      default: true
    protocol: TCP
  websecure:
    port: 8443          
    exposedPort: 443    
    expose:
      default: true
    protocol: TCP

gateway:
  enabled: true
  listeners:
    web:
      port: 8000        
      protocol: HTTP
      name: web
      namespacePolicy:
        from: All       
    websecure:
      port: 8443        
      protocol: HTTPS
      name: websecure
      namespacePolicy:
        from: All
      mode: Terminate
      certificateRefs:  
        - kind: Secret
          name: traefik-default-cert 
          group: ""

tlsStore:
  default:
    defaultCertificate:
      secretName: traefik-default-cert 
```

### Run Helm Installation
```bash
# Add and update the Traefik Helm repository
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Install Traefik using the validated chart version
helm install traefik traefik/traefik \
  --namespace traefik \
  --version 39.0.1 \
  -f values.yaml
```

---

## 🔑 Step 4: Grant MuleSoft RTF Agent Permissions
The RTF agent requires permission to manage `httproutes`. Ensure the namespace for the `ServiceAccount` matches your RTF deployment (e.g., `rtf` or `rtf-argo`).

### `rtf-rbac.yaml`
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rtf-agent-gateway-api-role
rules:
  - apiGroups:
      - "gateway.networking.k8s.io"
    resources:
      - "httproutes"
    verbs:
      - "get", "list", "watch", "create", "update", "patch", "delete"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rtf-agent-gateway-api-binding
subjects:
  - kind: ServiceAccount
    name: rtf-agent
    namespace: rtf-argo  
roleRef:
  kind: ClusterRole
  name: rtf-agent-gateway-api-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rtf-rbac.yaml
```

---

## 📜 Step 5: Define the HTTPRouteTemplate for RTF
The `HTTPRouteTemplate` dynamically generates routing objects when you deploy a Mule application.

### `rtf-template.yaml`
```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: traefik-gateway-template
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
          - matches:
              - path:
                  type: PathPrefix
                  value: {{ .Path }}
            backendRefs:
              - name: {{ .Service.Name }}
                port: {{ .Service.Port }}
```

```bash
kubectl apply -f rtf-template.yaml
```

---

## ✅ Final Verification

1.  **Check Gateway Status:** `kubectl get gateway traefik-gateway -n traefik`
    *Status should be `Programmed: True`.*
2.  **Access Traefik Dashboard:** `kubectl port-forward deployment/traefik -n traefik 9000:9000` 
    *Visit `http://localhost:9000/dashboard/`.*
3.  **Deploy Application:** In **Anypoint Runtime Manager**, select the endpoint under the **Ingress** tab during app deployment.
