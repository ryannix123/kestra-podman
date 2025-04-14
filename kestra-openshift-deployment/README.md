# Kestra on OpenShift Deployment Guide

This deployment package sets up **Kestra**, an open-source orchestration platform, on Red Hat OpenShift with:
- Persistent storage for Kestra, PostgreSQL, and Elasticsearch
- Secure HTTPS Route for the UI (with TLS edge termination and redirect)
- Non-root container configuration

---

## 📦 Included Files

- `values.yaml`: Helm configuration for non-root execution and persistent storage
- `pvc-kestra.yaml`: PVC definitions for Kestra, PostgreSQL, and Elasticsearch

---

## 🚀 Prerequisites

- OpenShift 4.x cluster
- `oc` and `helm` CLIs installed
- A storage class (e.g., `gp2`, `ocs-storagecluster-ceph-rbd`)
- Access to the OpenShift Web Console or API

---

## 🧰 Step-by-Step Installation

### 1. Create the OpenShift Project
```bash
oc new-project kestra
```

---

### 2. Create the PersistentVolumeClaims
```bash
oc apply -f pvc-kestra.yaml
```

---

### 3. (Optional) Customize Helm Values
Edit `values.yaml` if you want to make changes to storage or enable/disable route creation via Helm.

---

### 4. Add Kestra Helm Chart and Install
```bash
helm repo add kestra https://kestra-io.github.io/helm-charts
helm repo update

helm install kestra kestra/kestra \
  --namespace kestra \
  -f values.yaml
```

---

### 5. Create a Secure Route with TLS and Redirect
After installation, expose the service with TLS and force HTTPS:
```bash
oc expose svc kestra-ui --name=kestra-web --port=http
oc patch route kestra-web --type=merge -p '{"spec":{"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}'
```

Then view the route:
```bash
oc get route kestra-web
```

Open the HTTPS link in your browser to access Kestra.

---

## ✅ Notes

- Containers run as UID 1000 (non-root) and are OpenShift SCC compliant.
- For production, consider using external PostgreSQL and Elasticsearch services or operators.

---

## 🧹 Uninstall

```bash
helm uninstall kestra -n kestra
oc delete pvc kestra-storage kestra-postgres kestra-elasticsearch -n kestra
oc delete route kestra-web -n kestra
```

---

## 🔗 Resources

- [Kestra Docs](https://kestra.io/docs/)
- [Kestra GitHub](https://github.com/kestra-io/kestra)
