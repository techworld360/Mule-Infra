# Mule-Infra

Welcome to the **Mule-Infra** repository! This is a central hub for sharing detailed "How-To" guides, Kubernetes manifests, and configuration templates for **MuleSoft Infrastructure** setups.

The goal of this project is to provide the community with tested, production-ready patterns for managing Runtime Fabric (RTF) and Anypoint Platform components.

---

## 📂 Repository Contents

This repository contains step-by-step documentation and configuration files for:

* **RTF on Kubernetes:** Deploying Runtime Fabric on EKS, AKS, and GKE.
* **Service Mesh & Ingress:** Integrating **Istio** with the **K8s Gateway API**.
* **Security:** Managing TLS certificates and Kubernetes Secrets for Mule applications.
* **RBAC & Permissions:** Fine-grained access control for RTF agents and namespaces.

---

## 🚀 Getting Started

To implement any of the guides found in this repo:

1. **Clone the Repository:**
```bash
git clone https://github.com/techworld360/Mule-Infra.git

```


2. **Navigate to the Guide:**
Browse the directories to find the specific infrastructure component you wish to configure.
3. **Apply Configurations:**
Each directory contains a `README.md` with specific `kubectl` or `istioctl` commands.

---

## 🛠 Prerequisites

Before using these templates, ensure you have:

* A running Kubernetes cluster (v1.24+ recommended for Gateway API).
* [kubectl](https://kubernetes.io/docs/tasks/tools/) installed and configured.
* [Anypoint Platform](https://anypoint.mulesoft.com/) credentials with permissions to manage Runtime Fabric.

---

## 🤝 Community & Support

For any clarifications or if you run into issues while following the guides, please reach out.

> **Note:** Please expect a slight delay in response as we support the community in our spare time.

* **Discord:** Join the conversation on our [Discord Server](https://discord.gg/9Z8KfyqX).
* **Email:** Reach us directly at [techworld360@hotmail.com](mailto:techworld360@hotmail.com).

---

**Disclaimer:** *The configurations provided here are templates. Please review and test all YAML files in a sandbox environment before applying them to a production cluster.*
