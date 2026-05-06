# The Right Architecture — Go + MicroK8s + OpenBao + LXD

> Secrets architecture for a Go app running in MicroK8s that connects to LXD using credentials fetched from OpenBao at runtime.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Step 1 — Store OpenBao Credentials in K8s Secrets](#step-1--store-openbao-credentials-in-k8s-secrets)
- [Step 2 — Inject into Pod via Helm values.yaml](#step-2--inject-into-pod-via-helm-valuesyaml)
- [Step 3 — Go App Authenticates to OpenBao and Fetches LXD Secrets](#step-3--go-app-authenticates-to-openbao-and-fetches-lxd-secrets)
- [Step 4 — Store LXD Secrets in OpenBao](#step-4--store-lxd-secrets-in-openbao)
- [Full Picture](#full-picture)
- [Build and Deploy to MicroK8s](#build-and-deploy-to-microk8s)

---

## Architecture Overview

```
Kubernetes Secrets          OpenBao
(bootstrap credentials)     (actual secrets)
│                           │
├── VAULT_ADDR              ├── LXD_CLIENT_CERT
├── VAULT_ROLE_ID           └── LXD_CLIENT_KEY
└── VAULT_SECRET_ID

        │
        ↓
    Go Pod starts
        │
        ↓
    Go app reads K8s env vars
        │
        ↓
    Go app authenticates to OpenBao
        │
        ↓
    OpenBao returns LXD_CLIENT_CERT + LXD_CLIENT_KEY
        │
        ↓
    Go app connects to LXD
```

### Simple Rule

```
K8s Secrets   →  bootstrap credentials only (how to reach OpenBao)
OpenBao       →  all actual secrets (LXD certs, API keys, passwords)
Go code       →  no secrets ever, only reads from env/OpenBao at runtime
```

---

## Step 1 — Store OpenBao Credentials in K8s Secrets

```bash
microk8s kubectl create namespace dev

microk8s kubectl create secret generic openbao-creds \
  --from-literal=VAULT_ADDR=https://10.248.42.125:8200 \
  --from-literal=VAULT_ROLE_ID=99f070cb-f0cc-cd3d-bc55-2ac2a23b90bc \
  --from-literal=VAULT_SECRET_ID=4e17b858-475e-cfa8-d03a-bbd2337c8450 \
  --namespace=dev
```

---

## Step 2 — Inject into Pod via Helm values.yaml

```yaml
# values.yaml
secret:
  name: openbao-creds
```

```yaml
# templates/deployment.yaml
spec:
  containers:
    - name: lxd-reader
      image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
      envFrom:
        - secretRef:
            name: {{ .Values.secret.name }}   # injects all three VAULT_ vars
```

---

## Step 3 — Go App Authenticates to OpenBao and Fetches LXD Secrets

```go
package main

import (
    "fmt"
    "os"
    vault "github.com/hashicorp/vault/api"
)

func getSecrets() (cert, key string, err error) {
    // reads from pod environment (injected by K8s Secret)
    vaultAddr := os.Getenv("VAULT_ADDR")
    roleID    := os.Getenv("VAULT_ROLE_ID")
    secretID  := os.Getenv("VAULT_SECRET_ID")

    // connect to OpenBao
    config := vault.DefaultConfig()
    config.Address = vaultAddr
    client, err := vault.NewClient(config)
    if err != nil {
        return "", "", err
    }

    // authenticate with AppRole
    data := map[string]interface{}{
        "role_id":   roleID,
        "secret_id": secretID,
    }
    resp, err := client.Logical().Write("auth/approle/login", data)
    if err != nil {
        return "", "", err
    }
    client.SetToken(resp.Auth.ClientToken)

    // fetch LXD secrets from OpenBao
    secret, err := client.Logical().Read("secret/data/lxd")
    if err != nil {
        return "", "", err
    }

    cert = secret.Data["data"].(map[string]interface{})["LXD_CLIENT_CERT"].(string)
    key  = secret.Data["data"].(map[string]interface{})["LXD_CLIENT_KEY"].(string)

    return cert, key, nil
}
```

Go module setup:

```bash
go mod init go-lxd-vm-dashboard
go get github.com/hashicorp/vault/api
go get github.com/canonical/lxd/client
go get github.com/canonical/lxd/shared/api
```

---

## Step 4 — Store LXD Secrets in OpenBao

```bash
# Run on OpenBao server
bao kv put secret/lxd \
  LXD_CLIENT_CERT=@/path/to/client.crt \
  LXD_CLIENT_KEY=@/path/to/client.key
```

---

## Full Picture

```
Git / Helm chart
└── No secrets ever ✅

Kubernetes Secrets (microk8s)
└── VAULT_ADDR, VAULT_ROLE_ID, VAULT_SECRET_ID  (bootstrap only)

OpenBao
└── LXD_CLIENT_CERT, LXD_CLIENT_KEY  (actual sensitive secrets)

Go Pod
├── Reads VAULT_ vars from K8s env
├── Authenticates to OpenBao
└── Gets LXD certs at runtime → connects to LXD
```

This matches exactly how your Gitea Actions pipeline works — org-level vars for bootstrap, OpenBao for the real secrets.

---

## Build and Deploy to MicroK8s

```bash
# Build the Docker image
docker build -t go-lxd-vm-dashboard:latest .

# Load the image into MicroK8s
docker save go-lxd-vm-dashboard:latest > go-lxd-vm-dashboard.tar
microk8s ctr image import go-lxd-vm-dashboard.tar

# Create namespace and secrets
microk8s kubectl create namespace dev

microk8s kubectl create secret generic openbao-creds \
  --from-literal=VAULT_ADDR=https://10.248.42.125:8200 \
  --from-literal=VAULT_ROLE_ID=99f070cb-f0cc-cd3d-bc55-2ac2a23b90bc \
  --from-literal=VAULT_SECRET_ID=4e17b858-475e-cfa8-d03a-bbd2337c8450 \
  --namespace=dev

# Deploy via Helm
microk8s helm upgrade --install go-lxd-vm-dashboard ./helm-chart --namespace dev

# Force restart to refresh environment variables
microk8s kubectl rollout restart deployment go-lxd-vm-dashboard -n dev
```
