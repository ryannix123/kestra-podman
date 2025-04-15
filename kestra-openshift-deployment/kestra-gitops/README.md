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

---

## 📈 Scaling the Kestra Server with HPA (Horizontal Pod Autoscaler)

You can scale the Kestra server automatically based on CPU or memory usage using Kubernetes HPA.

### Example: HPA for Kestra Server

```bash
oc autoscale deployment kestra-standalone \
  --namespace=your-ocp-namespace \
  --cpu-percent=70 \
  --min=1 \
  --max=3
```

This creates an HPA that:
- Monitors CPU usage on the `kestra-standalone` deployment
- Scales pods between 1 and 3 replicas based on 70% CPU threshold

> ⚠️ PostgreSQL (running via Bitnami Helm chart) is **not horizontally scalable**. Use managed Postgres (e.g. CrunchyDB or RDS) for production scaling.

You can view the status of the autoscaler using:

```bash
oc get hpa -n your-ocp-namespace
```
