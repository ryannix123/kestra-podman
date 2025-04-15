# 📦 Kestra GitOps Deployment for OpenShift

This repo contains the GitOps resources needed to deploy Kestra on OpenShift using ArgoCD and the official Helm chart.

---

## 🗂 Folder Structure

- `values-openshift.yaml`: OpenShift-specific Helm overrides
- `kestra-argocd-app.yaml`: ArgoCD `Application` definition for automatic deployment

---

## 🚀 Deploy to ArgoCD

```bash
oc apply -f kestra-argocd-app.yaml -n openshift-gitops
```

## 🌍 Access

Once synced, expose the Kestra service with a TLS route:

```bash
oc expose svc/kestra-service -n your-ocp-namespace
```

## 🔄 Sync & Upgrade

ArgoCD will automatically sync updates to this repo and keep your deployment in sync.

For Helm version upgrades, just bump the tag in `values-openshift.yaml`:

```yaml
image:
  tag: v0.XX.X
```
