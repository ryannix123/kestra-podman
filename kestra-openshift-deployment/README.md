# Kestra on OpenShift Deployment Guide

This deployment package sets up **Kestra**, an open-source orchestration platform, on Red Hat OpenShift with:
- Persistent storage for Kestra, PostgreSQL, and Elasticsearch
- Secure HTTPS Route for the UI (with TLS edge termination and redirect)
- Non-root container configuration
- ✅ Docker sidecar disabled for OpenShift compatibility

---

## 📦 Included Files

- `values.yaml`: Helm configuration for OpenShift compliance and Docker disabled
- `pvc-kestra.yaml`: PVC definitions (namespace-agnostic)
- `README.md`: This guide

---

## 🚀 Prerequisites

- OpenShift 4.x cluster (Developer Sandbox compatible)
- `oc` and `helm` CLIs installed
- A storage class (e.g., `gp2`, `ocs-storagecluster-ceph-rbd`)
- An available namespace (e.g., `ryan-nix-dev` or `<your-namespace>`)

---

## 🧰 Step-by-Step Installation

### 1. Use or Create Your Project
```bash
oc project <your-namespace>
```

---

### 2. Apply the PersistentVolumeClaims
```bash
oc apply -f pvc-kestra.yaml -n <your-namespace>
```

---

### 3. Deploy Kestra with Helm
```bash
helm repo add kestra https://kestra-io.github.io/helm-charts
helm repo update

helm install kestra kestra/kestra \
  --namespace <your-namespace> \
  -f values.yaml
```

---

### 4. Create a Secure Route with TLS and Redirect
```bash
oc expose svc kestra-ui --name=kestra-web --port=http -n <your-namespace>
oc patch route kestra-web -n <your-namespace> --type=merge -p '{"spec":{"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}'
```

Then:
```bash
oc get route kestra-web -n <your-namespace>
```

Open the HTTPS URL in your browser.

---

## ✅ Notes

- No `runAsUser` or `privileged` settings are used — compliant with restricted SCC
- Docker sidecar (`dind`) is disabled for compatibility with OpenShift Developer Sandbox
- Works on OpenShift Developer Sandbox or full clusters

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
