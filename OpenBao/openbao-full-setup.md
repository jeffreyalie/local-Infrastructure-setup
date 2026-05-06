# OpenBao Full Working Setup (TLS + Raft + Web UI)

---

## Table of Contents

- [1. Required Directories](#1-required-directories)
- [2. Create CA (Self-Signed)](#2-create-ca-self-signed)
- [3. Create SAN Config](#3-create-san-config)
- [4. Generate Server Key](#4-generate-server-key)
- [5. Create CSR](#5-create-csr)
- [6. Sign Certificate](#6-sign-certificate)
- [7. Verify Files](#7-verify-files)
- [8. OpenBao Config](#8-openbao-config)
- [9. Fix systemd](#9-fix-systemd)
- [10. Start OpenBao](#10-start-openbao)
- [11. Set Local Access on Client Machine](#11-set-local-access-on-client-machine)
- [12. Initialize OpenBao](#12-initialize-openbao-only-once)
- [13. Unseal](#13-unseal)
- [14. Check Status](#14-check-status)
- [15. Access Web UI](#15-access-web-ui)
- [16. Fix Unknown Authority Error](#16-fix-unknown-authority-error-optional)
- [Phase 5 — AppRole Auth for Gitea Runner](#phase-5--approle-auth-for-gitea-runner)
- [Phase 6 — Update Gitea Org Secrets](#phase-6--update-gitea-org-secrets)
- [Phase 7 — Update GHA Workflows](#phase-7--update-gha-workflows)
- [Key Problems Fixed](#key-problems-fixed)
- [Final Mental Model](#final-mental-model)

---

## 1. Required Directories

```bash
mkdir -p /opt/openbao/{config,data,tls}
```

Config layout:

```
/opt/openbao/
├── config/
│    └── openbao.hcl
├── data/        (raft storage)
└── tls/
     ├── cert.pem
     ├── key.pem
     └── ca.pem
```

---

## 2. Create CA (Self-Signed)

```bash
cd /opt/openbao/tls

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 -days 3650 \
  -out ca.crt \
  -subj "/CN=openbao-ca"
```

---

## 3. Create SAN Config

> This avoids `req_ext` / object identifier errors.

```bash
vi san.cnf
```

Paste:

```ini
[req]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[dn]
CN = openbao.local

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = openbao.local
IP.1 = 10.248.42.125
```

---

## 4. Generate Server Key

```bash
openssl genrsa -out server.key 2048
```

---

## 5. Create CSR

```bash
openssl req -new \
  -key server.key \
  -out server.csr \
  -config san.cnf
```

---

## 6. Sign Certificate

```bash
openssl x509 -req \
  -in server.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out server.crt \
  -days 365 \
  -sha256 \
  -extfile san.cnf \
  -extensions req_ext
```

---

## 7. Verify Files

```bash
ls -l /opt/openbao/tls
```

You must have:

```
ca.crt
ca.key
server.key
server.crt   ✅
san.cnf
```

---

## 8. OpenBao Config

```bash
vi /opt/openbao/config/openbao.hcl
```

Paste:

```hcl
ui = true
disable_mlock = true

storage "raft" {
  path    = "/opt/openbao/data"
  node_id = "node1"
}

listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/opt/openbao/tls/server.crt"
  tls_key_file  = "/opt/openbao/tls/server.key"
}

api_addr     = "https://openbao.local:8200"
cluster_addr = "https://10.248.42.125:8201"
```

---

## 9. Fix systemd

```bash
vi /etc/systemd/system/openbao.service
```

Ensure it contains:

```ini
ExecStart=/usr/local/bin/openbao server -config=/opt/openbao/config/openbao.hcl
```

Then:

```bash
systemctl daemon-reload
```

---

## 10. Start OpenBao

```bash
systemctl enable openbao
systemctl restart openbao
systemctl status openbao
```

---

## 11. Set Local Access on Client Machine

Run this on your **laptop or runner** (NOT the server):

```bash
echo "10.248.42.125 openbao.local" >> /etc/hosts
```

---

## 12. Initialize OpenBao (Only Once)

```bash
export BAO_ADDR=https://openbao.local:8200
openbao operator init
```

Save the output carefully — you get unseal keys and a root token.

---

## 13. Unseal

```bash
openbao operator unseal
```

Repeat 3 times with different keys.

---

## 14. Check Status

```bash
openbao status
```

Expected:

```
Sealed: false
Active: true
```

---

## 15. Access Web UI

Open in browser:

```
https://openbao.local:8200
```

> Browser warning is normal for a self-signed cert. Click **Advanced → Proceed**.

---

## 16. Fix Unknown Authority Error (Optional)

On client machine:

```bash
cp /opt/openbao/tls/ca.crt /usr/local/share/ca-certificates/openbao.crt
update-ca-certificates
```

---

## Phase 5 — AppRole Auth for Gitea Runner

AppRole gives the runner a **Role ID + Secret ID** pair instead of a long-lived token.

```bash
# Enable AppRole auth
bao auth enable approle

# Create a policy for the runner
bao policy write gitea-runner - <<EOF
path "homelab/data/lxd" {
  capabilities = ["read"]
}
path "homelab/data/minio" {
  capabilities = ["read"]
}
path "homelab/data/ansible" {
  capabilities = ["read"]
}
EOF

# Create the AppRole
bao write auth/approle/role/gitea-runner \
  token_policies="gitea-runner" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=0   # 0 = never expires; tighten in prod

# Get the Role ID (goes into Gitea org secret)
bao read auth/approle/role/gitea-runner/role-id

# Generate a Secret ID (goes into Gitea org secret)
bao write -f auth/approle/role/gitea-runner/secret-id
```

---

## Phase 6 — Update Gitea Org Secrets

Replace 9 secrets with just 2:

| Old Secrets (9) | New Secrets (2) |
|-----------------|-----------------|
| LXD_ADDRESS, LXD_CLIENT_CERT, LXD_CLIENT_KEY, LXD_TRUST_PASSWORD, MINIO_ACCESS_KEY, MINIO_SECRET_KEY, MINIO_ENDPOINT, ANSIBLE_SSH_PRIVATE_KEY, ANSIBLE_SSH_PUBLIC_KEY | VAULT_ROLE_ID, VAULT_SECRET_ID |

In Gitea → Infra org → Settings → Secrets:

```
VAULT_ROLE_ID   = <role-id from above>
VAULT_SECRET_ID = <secret-id from above>
VAULT_ADDR      = https://openbao.local:8200   # or add as a 3rd secret
```

---

## Phase 7 — Update GHA Workflows

Add a **fetch-secrets** step at the top of every job:

```yaml
jobs:
  terraform:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - name: Fetch secrets from OpenBao
        env:
          VAULT_ADDR: ${{ secrets.VAULT_ADDR }}
          VAULT_ROLE_ID: ${{ secrets.VAULT_ROLE_ID }}
          VAULT_SECRET_ID: ${{ secrets.VAULT_SECRET_ID }}
        run: |
          # Login and get a short-lived token
          VAULT_TOKEN=$(curl -sf "$VAULT_ADDR/v1/auth/approle/login" \
            --data "{\"role_id\":\"$VAULT_ROLE_ID\",\"secret_id\":\"$VAULT_SECRET_ID\"}" \
            | jq -r '.auth.client_token')

          # Read secrets
          LXD=$(curl -sf -H "X-Vault-Token: $VAULT_TOKEN" \
            "$VAULT_ADDR/v1/homelab/data/lxd" | jq -r '.data.data')
          MINIO=$(curl -sf -H "X-Vault-Token: $VAULT_TOKEN" \
            "$VAULT_ADDR/v1/homelab/data/minio" | jq -r '.data.data')
          ANSIBLE=$(curl -sf -H "X-Vault-Token: $VAULT_TOKEN" \
            "$VAULT_ADDR/v1/homelab/data/ansible" | jq -r '.data.data')

          # Export to GITHUB_ENV
          echo "LXD_ADDRESS=$(echo $LXD | jq -r '.address')"                   >> $GITHUB_ENV
          echo "LXD_CLIENT_CERT=$(echo $LXD | jq -r '.client_cert')"           >> $GITHUB_ENV
          echo "LXD_CLIENT_KEY=$(echo $LXD | jq -r '.client_key')"             >> $GITHUB_ENV
          echo "TF_VAR_minio_access_key=$(echo $MINIO | jq -r '.access_key')"  >> $GITHUB_ENV
          echo "TF_VAR_minio_secret_key=$(echo $MINIO | jq -r '.secret_key')"  >> $GITHUB_ENV
          echo "MINIO_ENDPOINT=$(echo $MINIO | jq -r '.endpoint')"             >> $GITHUB_ENV

          # Write SSH keys to files
          echo "$(echo $ANSIBLE | jq -r '.ssh_private_key')" > /tmp/ansible_id_rsa
          echo "$(echo $ANSIBLE | jq -r '.ssh_public_key')"  > /tmp/ansible_id_rsa.pub
          chmod 600 /tmp/ansible_id_rsa

      - name: Terraform Init
        working-directory: terraform/env/dev
        run: terraform init

      - name: Ansible Playbook
        run: ansible-playbook -i ansible/inventory.yml ansible/playbook.yml \
               --private-key /tmp/ansible_id_rsa
```

---

## Key Problems Fixed

| Problem | Fix |
|---------|-----|
| Missing `cert.pem` / `server.crt` | Proper SAN config + correct filenames |
| `openssl req_ext` errors | Correct `[req]` + `[req_ext]` structure |
| Connection refused | Service was crashing due to missing TLS file |
| `x509 unknown authority` | Normal until CA is installed on client |

---

## Final Mental Model

```
server.crt / server.key  →  OpenBao TLS
ca.crt                   →  trust anchor for clients
openbao.local            →  internal DNS mapping
8200                     →  HTTPS API + UI
```
