## Guide: Configuring Istio with RTF via IngressClass

### 1. Download and Install the Istio CLI

First, fetch the latest Istio release and configure your local environment.

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -

# Navigate to the package directory
ISTIO_DIR=$(ls -d istio-* | head -n 1)
cd $ISTIO_DIR

# Add istioctl to your PATH
export PATH=$PWD/bin:$PATH

```

### 2. Install Istio (Default Profile)

Install the Istio control plane and the ingress gateway.

```bash
istioctl install --set profile=default -y

```

**Verify the installation:**

```bash
kubectl get pods -n istio-system
kubectl get svc -n istio-system istio-ingressgateway

```

---

### 3. Enable Istio Sidecar Injection

Label the namespace where your RTF applications will be deployed to ensure the Istio proxy (sidecar) is automatically injected.

```bash
kubectl label namespace <rtf-app-namespace> istio-injection=enabled --overwrite

```

---

### 4. Create TLS Secrets

RTF requires the TLS certificate to be present in both the `rtf` namespace (for synchronization) and the `istio-system` namespace (for the gateway to terminate traffic).

**Note:** Replace `base64 encoded cert/key` with your actual encoded strings.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: rtf
  labels:
    rtf.mulesoft.com/synchronized: "true"
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
---
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: istio-system
  labels:
    rtf.mulesoft.com/synchronized: "true"
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>

```

---

### 5. Define the IngressClass

Create an `IngressClass` to tell Kubernetes that Istio should handle Ingress resources specifically tagged with the `istio` class.

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: istio
spec:
  controller: istio.io/ingress-controller

```

---

### 6. Apply HTTPRouteTemplates

RTF uses `HTTPRouteTemplates` to automatically generate Kubernetes Ingress resources when you deploy an API. If you have multiple domains (e.g., a base domain and a wildcard), you need a template for each.

#### Template for `istio.techworld360.io`

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: rtf-hrt-istio-ingress
  namespace: rtf
spec:
  baseEndpoints:
    - https://istio.techworld360.io
  resources:
    - |
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        ingressClassName: istio
        tls:
        - hosts:
          - {{ .Host }}
          secretName: tls-secret
        rules:
        - host: {{ .Host }}
          http:
            paths:
            - pathType: Prefix
              path: {{ .Path }}
              backend:
                service:
                  name: {{ .Service.Name }}
                  port:
                    name: {{ .Service.PortName }}

```

#### Template for `*.istio.techworld360.io`

```yaml
apiVersion: rtf.mulesoft.com/v1
kind: HTTPRouteTemplate
metadata:
  name: rtf-hrt-istio-ingress-wildcard
  namespace: rtf
spec:
  baseEndpoints:
    - https://*.istio.techworld360.io
  resources:
    - |
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: {{ .ResourceName }}
        namespace: {{ .Namespace }}
      spec:
        ingressClassName: istio
        tls:
        - hosts:
          - {{ .Host }}
          secretName: tls-secret
        rules:
        - host: {{ .Host }}
          http:
            paths:
            - pathType: Prefix
              path: {{ .Path }}
              backend:
                service:
                  name: {{ .Service.Name }}
                  port:
                    name: {{ .Service.PortName }}

```

---

### 7. Configure RBAC for RTF Agent

The RTF agent needs permission to interact with Istio resources to manage the traffic flow correctly.

```yaml
# istio-permissions-rtf.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: rtf-istio-gateway-manager
rules:
- apiGroups: ["networking.istio.io"]
  resources: ["gateways", "virtualservices"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# istio-permissions-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: rtf-istio-binding
subjects:
- kind: ServiceAccount
  name: rtf-agent
  namespace: rtf
roleRef:
  kind: ClusterRole
  name: rtf-istio-gateway-manager
  apiGroup: rbac.authorization.k8s.io

```

---

### 8. Verification

Deploy your application from the **Anypoint Runtime Manager UI**. Once deployed, verify the pods and the sidecar injection.

**Check the pods:**

```bash
kubectl get po -n <your-app-namespace>

```

You should see `READY 3/3` (or similar), indicating that the `istio-proxy` sidecar is running alongside the Mule runtime and the app-init containers.

> **Pro Tip:** If your pods aren't showing the extra container, double-check that the namespace label `istio-injection=enabled` was applied *before* the deployment.


### Troubleshooting Checklist for Istio + RTF

#### 1. Ingress Class & Controller

* **Is the Ingress resource created?** Run `kubectl get ingress -n <app-namespace>` to ensure RTF successfully generated the resource based on your `HTTPRouteTemplate`.
* **Is the class correct?** Ensure the `ingressClassName` in the generated Ingress matches the name of the `IngressClass` resource you created (`istio`).
* **Controller match:** Verify that the `istio-ingressgateway` is actually watching for that specific class.

#### 2. Sidecar Injection

* **Pod Readiness:** If you see `1/1` or `2/2` instead of `3/3` in your pod status, the sidecar wasn't injected.
* Check namespace labels: `kubectl get namespace <app-namespace> --show-labels`.
* **Note:** Existing pods must be deleted and recreated (`kubectl delete pod ...`) after labeling a namespace for the sidecar to appear.



#### 3. Mutual TLS (mTLS) & Traffic

* **PeerAuthentication:** If you have a global mTLS policy set to `STRICT`, ensure RTF's internal health checks (which may come from outside the mesh) aren't being blocked.
* **VirtualServices:** Check if Istio automatically created a `VirtualService` for your Ingress. If not, the `istio-ingress-controller` might be having issues parsing the Ingress resource.

#### 4. Certificate & Secret Management

* **Secret Existence:** Ensure `tls-secret` exists in the `istio-system` namespace. The Istio Gateway cannot terminate HTTPS if it can't find the secret in its own namespace.
* **Sync Labels:** Verify the label `rtf.mulesoft.com/synchronized: "true"` is present so RTF recognizes the secret.

#### 5. Logs for Deep Debugging

If traffic isn't reaching your Mule app, check the logs in this order:

1. **Ingress Gateway Logs:** `kubectl logs -l istio=ingressgateway -n istio-system`
2. **App Sidecar Logs:** `kubectl logs <pod-name> -c istio-proxy -n <app-namespace>`
3. **RTF Agent Logs:** `kubectl logs -l app=agent -n rtf` (To see if it failed to process the `HTTPRouteTemplate`).
