# Install MicroK8s on Ubuntu

---

## Table of Contents

- [Step 1 — Install MicroK8s](#step-1--install-microk8s)
- [Step 2 — Enable Essential Add-ons](#step-2--enable-essential-add-ons)
- [Step 3 — Export Kubeconfig](#step-3--export-kubeconfig)
- [Step 4 — Install Lens](#step-4--install-lens)
- [Step 5 — Connect Lens to MicroK8s](#step-5--connect-lens-to-microk8s)
- [Common Issues and Fixes](#common-issues-and-fixes)
- [Fix Missing ~/.kube Directory](#fix-missing-kube-directory)
- [Kubernetes Dashboard](#kubernetes-dashboard)
- [Multiple Users and RBAC](#multiple-users-and-rbac)
- [Authentication Methods Comparison](#authentication-methods-comparison)

---

## Step 1 — Install MicroK8s

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install MicroK8s via Snap
sudo snap install microk8s --classic

# Add your user to the MicroK8s group (no sudo needed)
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
```

> Log out and back in for group changes to take effect.

```bash
# Check cluster status
microk8s status --wait-ready
```

---

## Step 2 — Enable Essential Add-ons

```bash
microk8s enable dns storage dashboard
```

| Add-on | Purpose |
|--------|---------|
| `dns` | CoreDNS for service discovery |
| `storage` | Default storage class |
| `dashboard` | Kubernetes dashboard |

You can also enable `ingress`, `metrics-server`, or `registry` depending on your needs.

---

## Step 3 — Export Kubeconfig

Lens requires kubeconfig to connect:

```bash
microk8s config > ~/.kube/config
```

> If you already have other clusters, merge configs carefully instead of overwriting.

---

## Step 4 — Install Lens

Download Lens from [https://k8slens.dev](https://k8slens.dev) and install it on Ubuntu:

```bash
sudo snap install kontena-lens --classic
# or use .deb package from their site

lens
```

---

## Step 5 — Connect Lens to MicroK8s

1. Open Lens → **Clusters** → **Add Cluster**
2. Select **Browse kubeconfig** and point to `~/.kube/config`
3. Lens will detect the MicroK8s cluster and add it
4. Click the cluster to connect — you'll see nodes, pods, services, and workloads

---

## Common Issues and Fixes

| Issue | Fix |
|-------|-----|
| Permissions error | Ensure your user is in the `microk8s` group and kubeconfig file is owned by you |
| Cluster not showing in Lens | Verify kubeconfig path and run `microk8s config` again |
| Networking issues | Enable `dns` and `ingress` add-ons |

---

## Fix Missing ~/.kube Directory

MicroK8s doesn't create `~/.kube` by default. Fix it manually:

```bash
# Step 1 — Create the directory
mkdir -p ~/.kube

# Step 2 — Export kubeconfig
microk8s config > ~/.kube/config

# Step 3 — Fix permissions
sudo chown -R $USER:$USER ~/.kube

# Step 4 — Verify
kubectl get nodes --kubeconfig ~/.kube/config
```

---

## Kubernetes Dashboard

After enabling the dashboard add-on:

```bash
# Port-forward the dashboard service
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

# Check service name if needed
kubectl -n kubernetes-dashboard get svc

# Get login token (if RBAC is not enabled)
microk8s kubectl describe secret -n kube-system microk8s-dashboard-token
```

Open [https://localhost:8443](https://localhost:8443) in your browser and paste the token.

---

## Multiple Users and RBAC

### Creating a Namespace and ServiceAccount

```bash
microk8s kubectl create namespace dev
microk8s kubectl -n dev create serviceaccount qa-user
```

### Binding to a ClusterRole (cluster-wide)

```bash
microk8s kubectl create clusterrolebinding qa-user-binding \
  --clusterrole=view \
  --serviceaccount=dev:qa-user
```

### Binding to a Namespace only

```bash
microk8s kubectl create rolebinding qa-user-binding \
  --clusterrole=view \
  --serviceaccount=dev:qa-user \
  --namespace=dev
```

### Generating a Token for the ServiceAccount

```bash
microk8s kubectl -n dev create token qa-user
```

### Example User Kubeconfig

```yaml
apiVersion: v1
kind: Config
clusters:
- name: microk8s-cluster
  cluster:
    server: https://127.0.0.1:16443
    certificate-authority-data: <CA_CERT_DATA>
users:
- name: qa-user
  user:
    token: <TOKEN_FROM_ABOVE>
contexts:
- name: qa-user-context
  context:
    cluster: microk8s-cluster
    user: qa-user
    namespace: dev
current-context: qa-user-context
```

---

## Authentication Methods Comparison

| Method | For | Expires | Revokable | Auditable | Setup |
|--------|-----|---------|-----------|-----------|-------|
| ServiceAccount token | Machines / CI-CD | Yes (short JWT) | Delete SA | ❌ No | Easy |
| Client Certificate | Humans | No (you set days) | Revoke cert | ✅ Yes (CN=name) | Medium |
| OIDC | Humans | Yes (auto-refreshes) | Revoke in provider | ✅ Yes (email) | Complex |

### Option A — Client Certificates (recommended for homelab)

```bash
# Step 1 — Generate private key
openssl genrsa -out jeffrey.key 2048

# Step 2 — Create CSR
openssl req -new \
  -key jeffrey.key \
  -out jeffrey.csr \
  -subj "/CN=jeffrey/O=dev-team"

# Step 3 — Sign with cluster CA
openssl x509 -req \
  -in jeffrey.csr \
  -CA /var/snap/microk8s/current/certs/ca.crt \
  -CAkey /var/snap/microk8s/current/certs/ca.key \
  -CAcreateserial \
  -out jeffrey.crt \
  -days 365

# Step 4 — Create RBAC
microk8s kubectl create clusterrolebinding jeffrey-binding \
  --clusterrole=view \
  --user=jeffrey

# Step 5 — Encode certs for kubeconfig
cat jeffrey.crt | base64 -w 0
cat jeffrey.key | base64 -w 0
```

### Option B — OIDC (enterprise)

```bash
# Configure API server
sudo nano /var/snap/microk8s/current/args/kube-apiserver
# Add:
# --oidc-issuer-url=https://accounts.google.com
# --oidc-client-id=my-kubectl-client
# --oidc-username-claim=email

sudo systemctl restart snap.microk8s.daemon-kubelite

# Install kubelogin
kubectl krew install oidc-login

# Test OIDC connection (opens browser, prints kubeconfig snippet)
kubectl oidc-login setup \
  --oidc-issuer-url=https://accounts.google.com \
  --oidc-client-id=my-kubectl-client

# Create RBAC (--user matches email claim)
microk8s kubectl create clusterrolebinding jeffrey-binding \
  --clusterrole=view \
  --user=jeffrey@gmail.com
```

> **Note:** `oidc-login setup` only tests + prints a snippet. You still build the kubeconfig manually.

### Key principle — Authentication vs Authorization

```
Step 1 → Authentication   "Who are you?"
         Certificate / OIDC / Token proves identity

Step 2 → Authorization    "What can you do?"
         RBAC always required — regardless of auth method
```
