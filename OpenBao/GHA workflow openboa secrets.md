GHA workflow openboa secrets

## Gite secretes flow

```
Gitea Org Secrets  
├── VAULT\_ADDR  
├── VAULT\_ROLE\_ID        → same for ALL workflows  
└── VAULT\_SECRET\_ID      → same for ALL workflows  
         │  
         ▼  
      OpenBao  
      (AppRole login)  
         │  
         ├── homelab/data/lxd        → LXD TLS cert/key    → go app talks to LXD  
         ├── homelab/data/minio      → MinIO creds          → Terraform state backend  
         ├── homelab/data/ansible    → Ansible secrets      → Ansible playbooks  
         └── homelab/data/microk8s   → kubeconfig          → helm deploy to MicroK8s
```
---

## Step 1 \- Setup Approl

**Enable KV v2 secrets engine:**

bash  
bao secrets enable \-path=homelab kv-v2

### **Phase 5 — AppRole Auth for the Gitea Runner**

AppRole gives the runner a **Role ID \+ Secret ID** pair instead of a long-lived token. The runner exchanges these for a short-lived token at the start of each job.

bash  
\# Enable AppRole auth  
bao auth enable approle

\# Create a policy for the runner  
openbao policy write gitea-runner \- \<\<'EOF'  
path "homelab/data/lxd" {  
  capabilities \= \["read"\]  
}  
path "homelab/data/minio" {  
  capabilities \= \["read"\]  
}  
path "homelab/data/ansible" {  
  capabilities \= \["read"\]  
}  
path "homelab/data/microk8s" {  
  capabilities \= \["read"\]  
}  
EOF

\# Create the AppRole  
bao write auth/approle/role/gitea-runner \\  
  token\_policies\="gitea-runner" \\  
  token\_ttl\=1h \\  
  token\_max\_ttl\=4h \\  
  secret\_id\_ttl\=0   \# 0 \= never expires; tighten this in prod

\# Get the Role ID (static, goes into Gitea org secret)  
bao read auth/approle/role/gitea-runner/role-id

\# Generate a Secret ID (also goes into Gitea org secret)  
bao write \-f auth/approle/role/gitea-runner/secret-id  
This will generate `VAULT_ROLE_ID`, `VAULT_SECRET_ID that runner need to connect to openbau to get other secrets.`  
—---------------------------------------------------------------------

**Step 2:**

**Write all your secrets into organised paths:**

bash  
\# LXD credentials  
bao kv put homelab/lxd \\  
  address\="https://10.x.x.x:8443" \\  
  client\_cert\="$(cat /path/to/lxd-client.crt)" \\  
  client\_key\="$(cat /path/to/lxd-client.key)" \\  
  trust\_password\="yourpassword"

\# MinIO credentials  
bao kv put homelab/minio \\  
  access\_key\="youraccesskey" \\  
  secret\_key\="yoursecretkey" \\  
  endpoint\="http://10.248.42.22:9000"

\# Ansible SSH keys  
bao kv put homelab/ansible \\  
  ssh\_private\_key\="$(cat \~/.ssh/id\_rsa)" \\  
  ssh\_public\_key\="$(cat \~/.ssh/id\_rsa.pub)"

\#Microk8s (.kube/config)  
openbao kv put homelab/microk8s kubeconfig\=@/tmp/kube.yaml

Before that:  
**On microk8s server**  
cat \~/.kube/config  
Copy the entire output.  
**On OpenBao VM**  
Paste it into a file:  
vi /tmp/kube.yaml  
Store kubeconfig:

**Verify:**

bash  
bao kv get homelab/lxd  
bao kv get homelab/minio  
bao kv get homelab/ansible  
---

### **Step 3  — Update Gitea Org Secrets**

Replace your 9 secrets with just **2**:

| Old Secrets (9) | New Secrets (2) |
| ----- | ----- |
| `LXD_ADDRESS`, `LXD_CLIENT_CERT`, `LXD_CLIENT_KEY`, `LXD_TRUST_PASSWORD`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `MINIO_ENDPOINT`, `ANSIBLE_SSH_PRIVATE_KEY`, `ANSIBLE_SSH_PUBLIC_KEY` | `VAULT_ROLE_ID`, `VAULT_SECRET_ID` |

In Gitea → Infra org → Settings → Secrets:

VAULT\_ROLE\_ID   \= \<role-id from above\>  
VAULT\_SECRET\_ID \= \<secret-id from above\>  
VAULT\_ADDR      \= http://vault01.lxd:8200   \# or make this a 3rd secret

## —-----------------------------------------------

### 

