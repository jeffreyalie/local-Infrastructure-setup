## Openboa \- microk8s \- Go app secrets
---

## Running Pod (request time)  

```
K8s Secret (openbao-creds)         ← created manually via kubectl  
├── VAULT\_ADDR                         in each namespace (dev/staging/live)  
├── VAULT\_ROLE\_ID  
└── VAULT\_SECRET\_ID  
         │  
         │  injected via envFrom in deployment.yaml  
         │  
         ▼  
      Go App (main.go)  
      getSecrets() function  
         │  
         │  AppRole login on every HTTP request  
         ▼  
      OpenBao  
         │  
         └── homelab/data/lxd  
                  │  
                  ├── client\_cert    → LXD TLS auth  
                  └── client\_key     → LXD TLS auth  
                           │  
                           ▼  
                        LXD API  
                  https://10.0.0.162:8443  
                           │  
                           ▼  
                    Instance list  
                    (name, type, status, IPs)  
                           │  
                           ▼  
                      index.html  
                      rendered to browser

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

### Step 3 \- Running Pod (request time)

### created manually via kubectl in each namespace (dev/staging/live)

### Store OpenBao credentials in K8s Secrets

bash  
microk8s kubectl create namespace dev  
microk8s kubectl create namespace staging  
microk8s kubectl create namespace live

microk8s kubectl create secret generic openbao-creds \\  
  \--from-literal=VAULT\_ADDR=https://10.248.42.125:8200 \\  
\--from-literal=VAULT\_ROLE\_ID=99f070cb-f0cc-cd3d-bc55-2ac2a23b90bc \\  
\--from-literal=VAULT\_SECRET\_ID=4e17b858-475e-cfa8-d03a-bbd2337c8450 \\  
  \--namespace=dev

microk8s kubectl create secret generic openbao-creds \\  
  \--from-literal=VAULT\_ADDR=https://10.248.42.125:8200 \\  
\--from-literal=VAULT\_ROLE\_ID=99f070cb-f0cc-cd3d-bc55-2ac2a23b90bc \\  
\--from-literal=VAULT\_SECRET\_ID=4e17b858-475e-cfa8-d03a-bbd2337c8450 \\  
  \--namespace=staging

microk8s kubectl create secret generic openbao-creds \\  
  \--from-literal=VAULT\_ADDR=https://10.248.42.125:8200 \\  
\--from-literal=VAULT\_ROLE\_ID=99f070cb-f0cc-cd3d-bc55-2ac2a23b90bc \\  
\--from-literal=VAULT\_SECRET\_ID=4e17b858-475e-cfa8-d03a-bbd2337c8450 \\  
  \--namespace=live

If want to recreate delete first:

microk8s kubectl delete secret openbao-creds \-n dev  
microk8s kubectl delete secret openbao-creds \-n staging  
microk8s kubectl delete secret openbao-creds \-n live

Restart microk8s pods:  
microk8s kubectl rollout restart deployment/go-lxd-vm-dashboard \-n dev  
microk8s kubectl rollout restart deployment/go-lxd-vm-dashboard \-n staging  
