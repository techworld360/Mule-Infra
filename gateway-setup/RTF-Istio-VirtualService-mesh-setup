
# Setup Mulesoft Runtime Fabric with Native Istio Virtual Service (for Service Mesh)

Setting up **MuleSoft Runtime Fabric (RTF)** with a **Native Istio Virtual Service** is a powerful way to leverage service mesh capabilities for your Mule applications.

---

## Architecture Overview

The following diagram illustrates the traffic flow from the Istio Ingress Gateway to your Mule application pod via Istio resources.

```text
       ┌──────────────────────────────┐
       │     istio-ingressgateway     │
       │     (Envoy proxy in pod)     │
       └──────────────┬───────────────┘
                      │
                      ▼
               Gateway resource
(Defines hostnames, ports, TLS certs, protocol)
                      │
                      ▼
           VirtualService resource
(Defines routing rules — path → service mapping)
                      │
                      ▼
           Kubernetes Service (app)
                      │
                      ▼
               Application Pod

```

---

## 1. Install and Configure Istio CLI

First, download the Istio CLI and set up your environment variables.

```bash
# Download and install Istio
curl -L https://istio.io/downloadIstio | sh -

# Move into the package directory
ISTIO_DIR=$(ls -d istio-* | head -n 1)
cd $ISTIO_DIR

# Add istioctl to your PATH
export PATH=$PWD/bin:$PATH

```

## 2. Install Istio in the Kubernetes Cluster

Install Istio using the default configuration profile.

```bash
istioctl install --set profile=default -y

```

## 3. Configure TLS Secrets

If you intend to use HTTPS, you must create a TLS Secret. This secret needs to be applied to both the **RTF namespace** and the **istio-system namespace** to ensure the ingress gateway can access the certificates.

### Apply to the `rtf` Namespace

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: istio-vs-tls
  namespace: rtf
  labels:
    rtf.mulesoft.com/synchronized: "true"
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
type: kubernetes.io/tls

```

### Apply to the `istio-system` Namespace

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: istio-vs-tls
  namespace: istio-system
  labels:
    rtf.mulesoft.com/synchronized: "true"
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
type: kubernetes.io/tls

```

---

## 4. Create the Istio Gateway

Create an Istio Gateway resource in the `istio-system` namespace. This gateway will reference the `istio-ingressgateway` and the TLS secret created in the previous step.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: istio-gwvs-tw360
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "*.istiovirtualservice.techworld360.io"
        - "istiovirtualservice.techworld360.io"
      tls:
        mode: SIMPLE
        credentialName: istio-vs-tls

```

**Verify the Gateway:**

```bash
kubectl get gw -n istio-system

```

---

## 5. Apply the HTTPRoute Templates

The `HTTPRouteTemplate` tells the RTF agent how to automatically create a `VirtualService` whenever a Mule application is deployed. Apply a separate template for each specific endpoint pattern.

### Template for Root Host

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: istio-virtualservice-hrt
  namespace: rtf
spec:
  baseEndpoints:
    - https://istiovirtualservice.techworld360.io
  resources:
    - |
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        hosts:
        - {{ .Host }}
        gateways:
        - istio-system/istio-gwvs-tw360
        http:
        - match:
          - uri:
              prefix: {{ .Path }}
          route:
          - destination:
              host: {{ .Service.Name }}
              port:
                number: {{ .Service.Port }}

```

### Template for Wildcard Host

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: istio-virtualservice-wildcard-hrt
  namespace: rtf
spec:
  baseEndpoints:
    - https://*.istiovirtualservice.techworld360.io
  resources:
    - |
      apiVersion: networking.istio.io/v1alpha3
      kind: VirtualService
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        hosts:
        - {{ .Host }}
        gateways:
        - istio-system/istio-gwvs-tw360
        http:
        - match:
          - uri:
              prefix: {{ .Path }}
          route:
          - destination:
              host: {{ .Service.Name }}
              port:
                number: {{ .Service.Port }}

```

---

## 6. Configure RBAC for the RTF Agent

The RTF agent requires permissions to manage Istio and Gateway API resources within your application namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rtf-agent-gateway-role-virtual-service
  namespace: rtf-appnamespace
rules:
  - apiGroups: ["gateway.networking.k8s.io", "networking.istio.io"]
    resources: ["httproutes", "gateways", "virtualservices"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rtf-agent-gateway-binding-virtualservice
  namespace: rtf-appnamespace
subjects:
  - kind: ServiceAccount
    name: rtf-agent
    namespace: rtf
roleRef:
  kind: Role
  name: rtf-agent-gateway-role-virtual-service
  apiGroup: rbac.authorization.k8s.io

```

---

## 7. Verification

After deploying your application from the Anypoint Runtime Manager UI, verify the status of the backend resources.

### A. Check Application Pods

```bash
kubectl get po -n rtf-appnamespace

```

### B. Check the Kubernetes Service

```bash
kubectl get svc -n rtf-appnamespace

```

### C. Check the VirtualService

Ensure that the `VirtualService` was successfully created by the RTF agent and points to the correct gateway and host.

```bash
kubectl get vs -n rtf-appnamespace

```

---

Would you like me to help you troubleshoot any specific errors you're seeing in the RTF agent logs?