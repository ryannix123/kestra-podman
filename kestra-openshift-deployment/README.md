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

- OpenShift 4.x cluster (Developer Sandbox compatible)
- `oc` and `helm` CLIs installed
- A storage class (e.g., `gp2`, `ocs-storagecluster-ceph-rbd`)
- An available namespace (e.g., `ryan-nix-dev` or `<your-namespace>`)

---

## 🧰 Step-by-Step Installation

### 1. Use or Create Your Project
If you're using the OpenShift Developer Sandbox, your namespace is likely fixed (e.g., `ryan-nix-dev`):
```bash
oc project <your-namespace>
```

If you're using your own cluster:
```bash
oc new-project <your-namespace>
```

---

### 2. Create the PersistentVolumeClaims
Edit the namespace in `pvc-kestra.yaml` if needed, then run:
```bash
oc apply -f pvc-kestra.yaml -n <your-namespace>
```

---

### 3. (Optional) Customize Helm Values
Edit `values.yaml` to configure route host or PVC names.

---

### 4. Add Kestra Helm Chart and Install
```bash
helm repo add kestra https://kestra-io.github.io/helm-charts
helm repo update

helm install kestra kestra/kestra \
  --namespace <your-namespace> \
  -f values.yaml
```

---

### 5. Create a Secure Route with TLS and Redirect
```bash
oc expose svc kestra-ui --name=kestra-web --port=http -n <your-namespace>
oc patch route kestra-web -n <your-namespace> --type=merge -p '{"spec":{"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}'
```

Then:
```bash
oc get route kestra-web -n <your-namespace>
```

Open the HTTPS link in your browser.

---

## ✅ Notes

- Containers run as UID 1000 (non-root) and are OpenShift SCC compliant.
- For production, consider using external PostgreSQL and Elasticsearch services or operators.

---

## 🧹 Uninstall

```bash
helm uninstall kestra -n <your-namespace>
oc delete pvc kestra-storage kestra-postgres kestra-elasticsearch -n <your-namespace>
oc delete route kestra-web -n <your-namespace>
```

---

## 🔗 Resources

- [Kestra Docs](https://kestra.io/docs/)
- [Kestra GitHub](https://github.com/kestra-io/kestra)
