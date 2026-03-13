# Istio with K8s Gateway API for Runtime Fabric (RTF)

This guide outlines the steps to configure Istio as the ingress controller for MuleSoft RTF using the Kubernetes Gateway API.

## 1. Environment Setup

Download and install the Istio CLI and set your environment variables.

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Move into the directory and update PATH
ISTIO_DIR=$(ls -d istio-* | head -n 1)
cd $ISTIO_DIR
export PATH=$PWD/bin:$PATH

```

## 2. Install Gateway API CRDs

The Gateway API resources (Gateway, HTTPRoute) are not always present by default in K8s clusters. Install the standard CRDs first:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml

```

## 3. Install Istio with Gateway API Support

Install Istio using the `default` profile, ensuring the Pilot component is configured to recognize Gateway API resources.

```bash
istioctl install --set profile=default \
  --set values.pilot.env.PILOT_ENABLE_GATEWAY_API=true

```

> **Note:** Setting `PILOT_ENABLE_GATEWAY_API=true` allows Istio's control plane (Pilot) to translate K8s Gateway objects into Envoy runtime configurations.

---

## 4. Configure TLS Secrets

If using HTTPS, create a TLS secret. For RTF to work correctly with Istio, the secret often needs to exist in the istio namespace (`istio-system`).

```bash

Store certificates in the `istio-system` namespace.


kubectl create secret tls istio-tls-secret \
  --namespace istio-system \
  --cert=path/to/fullchain.crt \
  --key=path/to/private.key

```

---

## 5. Create the Istio Gateway

Define the entry point for your traffic. This configuration covers both a wildcard subdomain and a root domain.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cluster-gateway
  namespace: istio-system
spec:
  gatewayClassName: istio
  listeners:
    - name: https-wildcard
      port: 443
      protocol: HTTPS
      hostname: "*.istiogateway.techworld360.io"
      tls:
        mode: Terminate
        certificateRefs:
          - name: istio-tls-secret
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              rtf.mulesoft.com/agentNamespace: rtf
    - name: https-root
      port: 443
      protocol: HTTPS
      hostname: "istiogateway.techworld360.io"
      tls:
        mode: Terminate
        certificateRefs:
          - name: testsecret-tls
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              rtf.mulesoft.com/agentNamespace: rtf

```

---

## 6. Apply HTTPRoute Template

RTF uses `HTTPRouteTemplate` to dynamically generate `HTTPRoute` objects when you deploy a Mule application.

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: template-gateway-wildcard
  namespace: rtf
spec:
  baseEndpoints:
    - https://*.istiogateway.techworld360.io
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
            name: cluster-gateway
            namespace: istio-system
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

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: template-gateway-apex
  namespace: rtf
spec:
  baseEndpoints:
    - https://istiogateway.techworld360.io
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
            name: cluster-gateway
            namespace: istio-system
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

## 7. Configure RBAC for RTF Agent

The RTF agent requires permissions to manage resources within the application namespaces.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: rtf-agent-gateway-role
  namespace: rtf-app-namespace # Ensure this matches your Mule App namespace
rules:
  - apiGroups: ["gateway.networking.k8s.io"]
    resources: ["httproutes", "gateways"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: rtf-agent-gateway-binding
  namespace: rtf-app-namespace
subjects:
  - kind: ServiceAccount
    name: rtf-agent
    namespace: rtf
roleRef:
  kind: Role
  name: rtf-agent-gateway-role
  apiGroup: rbac.authorization.k8s.io

```

---

## 8. Verification

After deploying your Mule application via Anypoint Runtime Manager, verify the resources:

**Check Application Pods:**

```bash
kubectl get po -n rtf-app-namespace

```

**Check Generated HTTPRoutes:**

```bash
kubectl get httproute -n rtf-app-namespace

```