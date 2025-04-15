# 🚀 Deploying Kestra on OpenShift (with Restricted SCCs)

This guide walks you through deploying [Kestra](https://kestra.io) on OpenShift using Helm, with all security configurations aligned to OpenShift's `restricted` SCC policy.

---

## ✅ Prerequisites

- OpenShift 4.x cluster
- Helm CLI installed
- A namespace with its own UID range (e.g., `ryan-nix-dev`)
- Internet access to pull images from Docker Hub

---

## 🔧 Step-by-Step Deployment

### 1. Add the Helm Repository

```bash
helm repo add kestra https://helm.kestra.io/
helm repo update
```

---

### 2. Configure `values-openshift.yaml`

Create a `values-openshift.yaml` file with these key settings:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1008630000
  fsGroup: 1008630000
  allowPrivilegeEscalation: false

postgresql:
  enabled: true
  auth:
    username: kestra
    password: k3str4
    database: kestra
  primary:
    securityContext:
      runAsNonRoot: true
      runAsUser: 1008630000
      fsGroup: 1008630000
      allowPrivilegeEscalation: false

dind:
  enabled: false
```

Update the `runAsUser` and `fsGroup` with your namespace’s UID range from:

```bash
oc get project ryan-nix-dev -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'
```

---

### 3. Deploy Kestra via Helm

```bash
helm install kestra kestra/kestra   --namespace ryan-nix-dev   --create-namespace   -f values-openshift.yaml
```

---

### 4. Expose Kestra with a TLS Route

Apply the provided OpenShift Route:

```bash
oc apply -f kestra-route.yaml
```

This creates a TLS-secured external route with automatic HTTP → HTTPS redirection.

---

### 5. Access the UI

Get the route:

```bash
oc get route kestra -n ryan-nix-dev -o wide
```

Visit the URL in your browser.

Default credentials:

- **Username:** `admin`
- **Password:** `admin`

---

## ✅ Done!

Your Kestra instance is now fully deployed with:

- Non-root SCC compliance
- PostgreSQL backend
- TLS-secured UI access
- Auto-migrated schema via Helm chart

For more, see: [https://kestra.io/docs](https://kestra.io/docs)

---

## 🔄 Upgrading to Future Versions of Kestra

To upgrade Kestra when a new version is released:

1. Update the Helm repo:
```bash
helm repo update
```

2. Edit your `values-openshift.yaml` to bump the image version if desired:
```yaml
image:
  tag: v0.XX.X
```

3. Upgrade the release:
```bash
helm upgrade kestra kestra/kestra   --namespace ryan-nix-dev   -f values-openshift.yaml
```

4. Monitor the rollout:
```bash
oc rollout status deployment/kestra-standalone -n ryan-nix-dev
```

> ⚠️ Always check [https://kestra.io/releases](https://kestra.io/releases) for breaking changes or migration notes before upgrading.
