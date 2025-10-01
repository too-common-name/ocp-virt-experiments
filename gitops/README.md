# OpenShift Virtualization GitOps

This repository provides a GitOps approach for managing Virtual Machines (VMs) in OpenShift using Argo CD and Kustomize. It enables declarative VM configuration and consistent deployment across environments.

## 📁 Repository Structure

- `apps/`: Argo CD application definitions.
- `templates/`: Generic YAML templates for VM definitions.

## 🚀 Prerequisites

- OpenShift cluster with OpenShift Virtualization enabled (version ≥ 4.18).
- Argo CD installed and configured.
- CLI tools: `oc`, `gh` (optional).

## 📘 Usage Guide

**NOTE:** The `secondary-net-mnp` setup is designed for a specific lab environment. Your setup might not require interface reconfiguration (meaning your `templates/secondary-net-mnp/nncp.yaml` might not need the remove rule). To use Multi-Network Policy, you must run `oc patch network.operator.openshift.io cluster --type=merge -p '{"spec":{"useMultiNetworkPolicy":true}}'` before creating the ArgoCD app.

1. Fork this repo into your own GitHub account using CLI or UI.

```bash
gh repo fork too-common-name/ocp-virt-gitops
```

2. Clone your fork

```bash
git clone https://github.com/<your-username>/ocp-virt-gitops.git
cd ocp-virt-gitops
```

3. Replace repository url of ArgoCD app manifests 

```bash
sed -i 's|https://github.com/too-common-name/ocp-virt-gitops.git|https://github.com/<your-username>/ocp-virt-gitops.git|g' apps/**/*.yaml
```
4. Create ArgoCD application
```bash
oc create -f apps
```

## ⚠️ Notes

- **This repository is for demonstration purposes only**. For simplicity, cloud-init `userData` is included directly in the VM manifest. In production, use encrypted secrets (e.g., via [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)) to protect sensitive configuration.
- **Argo CD autosync is disabled by default**. Changes pushed to Git must be synced manually via the Argo CD UI or CLI.
