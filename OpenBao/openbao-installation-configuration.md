# OpenBao — Installation and Configuration

> See also: [openbao-full-setup.md](./openbao-full-setup.md) for the complete TLS + Raft + Web UI setup (Phase 5 onwards are covered there).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Phase 1 — Install OpenBao](#phase-1--install-openbao)
- [Phase 2 — Configure OpenBao](#phase-2--configure-openbao)
- [Phase 3 — Initialize and Unseal](#phase-3--initialize-and-unseal)
- [Phase 4 — Store Your Secrets](#phase-4--store-your-secrets)
- [Phase 5 — AppRole Auth for Gitea Runner](#phase-5--approle-auth-for-gitea-runner)
- [Phase 6 — Update Gitea Org Secrets](#phase-6--update-gitea-org-secrets)
- [Phase 7 — Update GHA Workflows](#phase-7--update-gha-workflows)
- [Bonus — Auto-Unseal on Restart](#bonus--auto-unseal-on-restart-optional)
- [Summary of Changes](#summary-of-changes)

---

## Architecture Overview

OpenBao runs in a dedicated LXD VM (e.g., `vault01`). Your Gitea runner fetches secrets at job runtime via the OpenBao HTTP API using **AppRole** auth — replacing static org secrets entirely.

```
Gitea Actions Job
    │
    ├─ Step 1: Login to OpenBao (AppRole → short-lived token)
    ├─ Step 2: Read secrets from KV store
    └─ Step 3: Export as env vars → Terraform / Ansible use them normally
```

---

## Phase 1 — Install OpenBao

Spin up a dedicated LXD VM:

```bash
lxc launch ubuntu-24-04-vm vault01 --vm
lxc shell vault01
```

Inside the VM — install OpenBao:

```bash
sudo apt update && sudo apt install -y curl gpg

# Add OpenBao repo
curl -fsSL https://packages.openbao.org/openbao.gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/openbao.gpg

echo "deb [signed-by=/usr/share/keyrings/openbao.gpg] https://packages.openbao.org/ubuntu noble main" \
  | sudo tee /etc/apt/sources.list.d/openbao.list

sudo apt update && sudo apt install -y openbao
```

---

## Phase 2 — Configure OpenBao

Create `/etc/openbao/openbao.hcl`:

```hcl
ui            = true
disable_mlock = true   # safe for a VM; avoids needing extra kernel capability

storage "file" {
  path = "/opt/openbao/data"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true   # use Nginx TLS termination, or enable cert here for prod
}

api_addr     = "http://vault01.lxd:8200"
cluster_addr = "http://vault01.lxd:8201"
```

```bash
sudo mkdir -p /opt/openbao/data
sudo chown -R openbao:openbao /opt/openbao
sudo systemctl enable --now openbao
```

Set environment shorthand on the VM (add to `~/.bashrc`):

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
```

---

## Phase 3 — Initialize and Unseal

```bash
# Initialize — generates unseal keys and root token
bao operator init -key-shares=3 -key-threshold=2
```

> Save the output carefully — you get 3 unseal keys and 1 root token. You need **2 of the 3** unseal keys every time the service restarts.

```bash
# Unseal (run twice with different keys)
bao operator unseal <unseal-key-1>
bao operator unseal <unseal-key-2>

# Login with root token
bao login <root-token>

# Verify
bao status
```

---

## Phase 4 — Store Your Secrets

Enable KV v2 secrets engine:

```bash
bao secrets enable -path=homelab kv-v2
```

Write secrets into organised paths:

```bash
# LXD credentials
bao kv put homelab/lxd \
  address="https://10.x.x.x:8443" \
  client_cert="$(cat /path/to/lxd-client.crt)" \
  client_key="$(cat /path/to/lxd-client.key)" \
  trust_password="yourpassword"

# MinIO credentials
bao kv put homelab/minio \
  access_key="youraccesskey" \
  secret_key="yoursecretkey" \
  endpoint="http://10.248.42.22:9000"

# Ansible SSH keys
bao kv put homelab/ansible \
  ssh_private_key="$(cat ~/.ssh/id_rsa)" \
  ssh_public_key="$(cat ~/.ssh/id_rsa.pub)"
```

Verify:

```bash
bao kv get homelab/lxd
bao kv get homelab/minio
bao kv get homelab/ansible
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
VAULT_ADDR      = http://vault01.lxd:8200   # or add as a 3rd secret
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
          VAULT_TOKEN=$(curl -sf "$VAULT_ADDR/v1/auth/approle/login" \
            --data "{\"role_id\":\"$VAULT_ROLE_ID\",\"secret_id\":\"$VAULT_SECRET_ID\"}" \
            | jq -r '.auth.client_token')

          LXD=$(curl -sf -H "X-Vault-Token: $VAULT_TOKEN" \
            "$VAULT_ADDR/v1/homelab/data/lxd" | jq -r '.data.data')
          MINIO=$(curl -sf -H "X-Vault-Token: $VAULT_TOKEN" \
            "$VAULT_ADDR/v1/homelab/data/minio" | jq -r '.data.data')
          ANSIBLE=$(curl -sf -H "X-Vault-Token: $VAULT_TOKEN" \
            "$VAULT_ADDR/v1/homelab/data/ansible" | jq -r '.data.data')

          echo "LXD_ADDRESS=$(echo $LXD | jq -r '.address')"                   >> $GITHUB_ENV
          echo "LXD_CLIENT_CERT=$(echo $LXD | jq -r '.client_cert')"           >> $GITHUB_ENV
          echo "LXD_CLIENT_KEY=$(echo $LXD | jq -r '.client_key')"             >> $GITHUB_ENV
          echo "TF_VAR_minio_access_key=$(echo $MINIO | jq -r '.access_key')"  >> $GITHUB_ENV
          echo "TF_VAR_minio_secret_key=$(echo $MINIO | jq -r '.secret_key')"  >> $GITHUB_ENV
          echo "MINIO_ENDPOINT=$(echo $MINIO | jq -r '.endpoint')"             >> $GITHUB_ENV

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

## Bonus — Auto-Unseal on Restart (Optional)

OpenBao requires manual unsealing after every restart by default. For a homelab, you can use **Transit auto-unseal** (one OpenBao unsealing another), or a simpler approach: a small systemd `ExecStartPost` script that reads your unseal keys from an encrypted file on the host.

> Store unseal keys in a file readable only by root, and auto-unseal via a script called by systemd after OpenBao starts. Not production-grade but removes the manual step for a home lab.

---

## Summary of Changes

| Concern | Before | After |
|---------|--------|-------|
| Secret storage | Gitea org secrets (9) | OpenBao KV v2 |
| Secret rotation | Edit each Gitea secret manually | `bao kv put` updates all consumers instantly |
| Secret count in Gitea | 9 secrets | 2 secrets (ROLE_ID, SECRET_ID) |
| Audit trail | None | OpenBao audit log |
| Token lifetime | Forever (static) | 1 hour (per-job AppRole token) |
