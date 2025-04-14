# Kestra on OpenShift Deployment Guide

This deployment package sets up **Kestra**, an open-source orchestration platform, on Red Hat OpenShift with:
- Persistent storage for Kestra, PostgreSQL, and Elasticsearch
- Secure HTTPS Route for the UI
- Non-root container configuration

---

## 📦 Included Files

- `values.yaml`: Helm configuration for non-root execution, persistent storage, and secure route
- `pvc-kestra.yaml`: PVC definitions for Kestra, PostgreSQL, and Elasticsearch

---

## 🚀 Prerequisites

- OpenShift 4.x cluster
- `oc` and `helm` CLIs installed
- A storage class (e.g., `gp2`, `ocs-storagecluster-ceph-rbd`)
- Access to the OpenShift Web Console or API
- A valid domain/cluster route (replace `kestra.apps.YOUR-CLUSTER.DOMAIN` accordingly)

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

### 3. Update Helm Values (Optional)
Edit `values.yaml` and set the correct route host:
```yaml
route:
  host: kestra.apps.<your-cluster-domain>
```

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

### 5. Verify the Secure Route
```bash
oc get route kestra
```
Open the `https://` URL in your browser.

---

## ✅ Notes

- Containers run as UID 1000 (non-root) and are OpenShift SCC compliant.
- For production, use external PostgreSQL and Elasticsearch services or operators.

---

## 🧹 Uninstall

```bash
helm uninstall kestra -n kestra
oc delete pvc kestra-storage kestra-postgres kestra-elasticsearch -n kestra
```

---

## 🔗 Resources

- [Kestra Docs](https://kestra.io/docs/)
- [Kestra GitHub](https://github.com/kestra-io/kestra)

