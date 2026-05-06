# Secrets — Variables and Configuration for Automation

---

## Table of Contents

- [Where Secrets Are Used](#where-secrets-are-used)
- [Secret Variables Reference](#secret-variables-reference)
- [Secret Values](#secret-values)
- [How to Get LXD_CLIENT_CERT and LXD_CLIENT_KEY](#how-to-get-lxd_client_cert-and-lxd_client_key)
- [How to Get ANSIBLE_SSH_PUBLIC_KEY](#how-to-get-ansible_ssh_public_key)
- [How Secrets Flow into Terraform](#how-secrets-flow-into-terraform)
- [Important Points](#important-points)

---

## Where Secrets Are Used

**For GHA automation** (`.gitea/workflows/*.yml`):
- Add at org or repo level as Gitea secrets
- Define actual values in OpenBao

**For manual use:**
- Export all as environment variables in your shell

---

## Secret Variables Reference

| Variable | Description |
|----------|-------------|
| `LXD_ADDRESS` | LXD host IP address |
| `LXD_CLIENT_CERT` | LXD client certificate |
| `LXD_CLIENT_KEY` | LXD client private key |
| `MINIO_ACCESS_KEY` | MinIO admin username |
| `MINIO_SECRET_KEY` | MinIO admin password |
| `MINIO_ENDPOINT` | MinIO API server address |
| `ANSIBLE_SSH_PRIVATE_KEY` | SSH private key for Ansible |
| `ANSIBLE_SSH_PUBLIC_KEY` | SSH public key for Ansible |
| `LXD_TRUST_PASSWORD` | LXD trust password |

---

## Secret Values

| Variable | How to Get | Example Value |
|----------|-----------|---------------|
| `LXD_ADDRESS` | `ip -4 addr` | `10.0.0.162` |
| `LXD_CLIENT_CERT` | `cat ~/lxd-certs/gitea-runner.crt` | PEM certificate content |
| `LXD_CLIENT_KEY` | `cat ~/lxd-certs/gitea-runner.key` | PEM key content |
| `MINIO_ACCESS_KEY` | MinIO admin username | `minioadmin` |
| `MINIO_SECRET_KEY` | MinIO admin password | `minioadmin` |
| `MINIO_ENDPOINT` | MinIO API server address | `http://10.248.42.22:9000` |
| `ANSIBLE_SSH_PRIVATE_KEY` | `cat ~/.ssh/ansible_id_ed25519` | Private key content |
| `ANSIBLE_SSH_PUBLIC_KEY` | `cat ~/.ssh/ansible_id_ed25519.pub` | Public key content |
| `LXD_TRUST_PASSWORD` | Via LXD web UI | Set during LXD init |

---

## How to Get LXD_CLIENT_CERT and LXD_CLIENT_KEY

### Step 1 — Generate a Client Certificate

Run this on your LXD host:

```bash
# Create a directory for the cert
mkdir -p ~/lxd-certs && cd ~/lxd-certs

# Generate client cert + key (valid 10 years)
openssl req -x509 -newkey ec \
  -pkeyopt ec_paramgen_curve:secp384r1 \
  -sha384 -keyout gitea-runner.key \
  -out gitea-runner.crt \
  -days 3650 -nodes \
  -subj "/CN=gitea-runner"
```

### Step 2 — Trust the Certificate in LXD

```bash
# Add the cert as a trusted client
lxc config trust add ~/lxd-certs/gitea-runner.crt --name gitea-runner
```

Or via **LXD Web UI**: Settings → Trusted clients → Add certificate → paste `gitea-runner.crt` content.

Verify it's trusted:

```bash
lxc config trust list
```

### Step 3 — Add Secrets to Gitea

| Secret Name | Value | How to Get |
|-------------|-------|-----------|
| `LXD_CLIENT_CERT` | Content of `gitea-runner.crt` | `cat ~/lxd-certs/gitea-runner.crt` |
| `LXD_CLIENT_KEY` | Content of `gitea-runner.key` | `cat ~/lxd-certs/gitea-runner.key` |
| `LXD_ADDRESS` | Your host IP | `hostname -I \| awk '{print $1}'` |
| `ANSIBLE_SSH_PUBLIC_KEY` | Your ansible pub key | See section below |

---

## How to Get ANSIBLE_SSH_PUBLIC_KEY

### Step 1 — Generate Your SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "ansible" -f ~/.ssh/ansible_id_ed25519
# Press Enter twice for no passphrase (recommended for automation)
```

This creates two files:

- `~/.ssh/ansible_id_ed25519` — **private key** (keep safe, never share)
- `~/.ssh/ansible_id_ed25519.pub` — **public key** (this goes into Gitea)

### Step 2 — Copy the Public Key Content

```bash
cat ~/.ssh/ansible_id_ed25519.pub
# Output looks like:
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ansible
```

Copy that entire line.

### Step 3 — Open Your Gitea Repository

```
http://<your-gitea-host>/your-username/lxd-vm-terraform
```

### Step 4 — Navigate to Secrets

Settings → Secrets and Variables → Actions → New Secret

### Step 5 — Fill in the Form

| Field | Value |
|-------|-------|
| Name | `ANSIBLE_SSH_PUBLIC_KEY` |
| Value | Paste the full public key line |

Click **Add Secret**.

---

## How Secrets Flow into Terraform

In the workflow file (`.gitea/workflows/terraform.yml`), the secret is referenced as:

```yaml
env:
  TF_VAR_ansible_ssh_public_key: ${{ secrets.ANSIBLE_SSH_PUBLIC_KEY }}
```

**Flow:**

```
Gitea secret injected as env var at runtime
    ↓
Terraform picks it up as var.ansible_ssh_public_key
    ↓
Templated into cloud-init/user-data.yaml
    ↓
cloud-init writes it to /home/ansible/.ssh/authorized_keys inside the VM
```

---

## Important Points

- **Public key is not technically sensitive**, but keeping it in Gitea Secrets is clean practice and avoids cluttering your code.
- **Never put the private key** (`ansible_id_ed25519`) anywhere near Gitea or Terraform.
- The private key stays on the machine running Ansible playbooks against the VM:

```bash
ssh -i ~/.ssh/ansible_id_ed25519 ansible@<VM-IP>
```

- **Secrets are masked in pipeline logs** — Gitea will never print the value even if you accidentally echo it.
