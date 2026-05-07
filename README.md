# Homelab Infrastructure — Installation Guides

Documentation for the LXD · Gitea · Gitea Runner · OpenBao · MinIO · MicroK8s · Go homelab stack.

---

## Table of Contents

### LXD

- [Install LXD (Snap-based)](./LXD/install-lxd.md)
  - Install LXD, enable the Web UI, configure trust password, launch containers, enable remote access

### Gitea

- [Gitea on MicroK8s](./Gitea/gitea-on-microk8s.md)
  - Enable MicroK8s addons, add Helm chart, configure values.yaml, install and access Gitea

- [Docker Socket Mode Runner on Ubuntu 20.04](./Gitea/docker-socket-runner.md)
  - Register act_runner with Gitea, Docker Socket Mode, docker-compose setup, multiple runners

### MinIO

- [Install MinIO on a VM (Linux)](./Minio/install-minio.md)
  - Download binary, configure systemd service, set credentials, verify

### OpenBao

- [OpenBao Full Working Setup (TLS + Raft + Web UI)](./OpenBao/openbao-full-setup.md)
  - Self-signed CA, SAN config, TLS certificates, Raft storage, systemd fix, initialize, unseal, Web UI, AppRole auth, GHA workflow integration

- [OpenBao Installation and Configuration](./OpenBao/openbao-installation-configuration.md)
  - LXD VM setup, install from repo, file-based storage, KV v2 secrets, AppRole, Gitea org secrets migration, auto-unseal

### MicroK8s · Vault · Go on Pod

- [Install MicroK8s on Ubuntu](./microk8s-vault-go%20on%20pod/install-microk8s.md)
  - Install via snap, enable add-ons, export kubeconfig, Lens setup, Kubernetes Dashboard, RBAC, multiple user auth methods (ServiceAccount / Client Certificate / OIDC)

- [The Right Architecture — Go + MicroK8s + OpenBao + LXD](./microk8s-vault-go%20on%20pod/right-architecture.md)
  - K8s Secrets as bootstrap, OpenBao for actual secrets, Go AppRole auth, Helm deployment, build and import image to MicroK8s

### Secrets

- [Secrets — Variables and Configuration for Automation](./Secrets/for-automation.md)
  - All secret variable names and values, generate LXD client cert, generate Ansible SSH keys, add secrets to Gitea, flow into Terraform and cloud-init

---

## For LXD - Gitea - Gittea runner - OpenBao - Minio - VM deployment and Ansible

### Architecture Overview - (Infrastrucutre - GHA workflow - Secrets workflow)

```
                ┌─────────────────────────────────────────────┐
                │           Gitea (gitea.local)               │  (Org level secrets for OpenBao)
                │   local-workflows-ansible-roles-modules     │
                └─────────────────────┬───────────────────────┘
                                      │ Gitea Actions triggers                                              
                                      ▼                                                     ┌───────────────┐ 
                        ┌─────────────────────────┐                                         │    OpenBao    │ 
                        │     Gitea Act Runner    │  (Docker-based, inside LXD VM) ─────────│   (secrets)   │ 
                        └─────┬─────────┬─────────┘                                         │ For LXD /Minio│ 
                              │         │                                                   └───────────────┘  
                              │ calls   │ calls                                                   
                              ▼         ▼
          ┌──────────────────────────────────────────────────────────┐
          │                  Shared Gitea Repos (Infra org)          │
          │                                                          │
          │  reusable-workflows-vault   ← Default  (OpenBao secrets) │
          │  reusable-workflows         ← Alternate (org secrets)    │
          │  reusable-modules           ← Terraform lxd-vm module    │
          │  reusable-ansible-galaxy-*  ← Ansible Galaxy roles       │
          └──────────────────────────────────────────────────────────┘
                       │                             │
              Terraform│                             │Ansible
                       ▼                             ▼
              ┌─────────────────┐           ┌─────────────────┐
              │      MinIO      │           │   Target VM     │
              │    backend s3   │           │  (Ubuntu 24.04) │
              │    (TF state)   │           │  ansible user   │
              └─────────────────┘           └─────────────────┘
                      │                    
                      │ Creates VM
                      │ 
              ┌─────────────────┐ 
              │   LXD / KVM     │ 
              │  (localhost:    │ 
              │    8443)        │ 
              └─────────────────┘  
```

---

## For Microk8s - Go - Gitea - Gitea runner - GHA - OpenBao

### Architecture - (Infrastrucutre - Go app internal workflow - Go app internal Secrets workflow)

```
                              ┌────────────────────┐
                              │     index.html     │
                              └────────────────────┘
                                      │   ▲
                                      │   │  
                                      ▼   │ 
                              ┌────────────────────┐
                              │      Browser       │
                              └─────────┬──────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │  Ingress (NGINX)   │
                              └─────────┬──────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │      Service       │  ---- Go app gets internal secreat from openbao for lxd
                              └─────────┬──────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │   Pod (MicroK8s)   │  (OpenBao secrets for OpenBao)
                              └─────────┬──────────┘
                                        │
                  ┌─────────────────────┴─────────────────────┐
                  │                                           │
                  ▼                                           ▼
      ┌──────────────────────────────┐         ┌──────────────────────────────┐
      │ Fetch TLS cert/key           │         │ Query LXD API                │
      │ from OpenBao (AppRole)       │         │ (LXD Server)                 │
      └──────────────────────────────┘         └──────────────────────────────┘

```

---

### Architecture - (Infrastrucutre - GHA workflow - Secrets workflow)

```
          ┌──────────────────────────────────────────────────┐
          │              Gitea Org Secrets                   │
          │  ├── VAULT_ADDR                                  │
          │  ├── VAULT_ROLE_ID      → same for ALL workflows │
          │  └── VAULT_SECRET_ID                             │
          └──────────────────────────┬───────────────────────┘
                                    │
                                    ▼
          ┌──────────────────────────────────────────────────┐
          │                   OpenBao                        │  -----------------  Get .kube/config info from openbau
          │               AppRole login                      │
          └──────────────────────────┬───────────────────────┘
                                    │
                                    ▼
          ┌──────────────────────────────────────────────────┐
          │         homelab/data/microk8s                    │
          │              kubeconfig                          │
          │   (server: https://10.0.0.162:16443)             │
          └──────────────────────────┬───────────────────────┘
                                    │
                                    ▼
          ┌──────────────────────────────────────────────────┐
          │           GHA Runner (LXD VM)                    │
          │                                                  │
          └──────────┬───────────────┬───────────────┬───────┘
                    │               │               │
                    ▼               ▼               ▼
              ┌────────────┐  ┌────────────┐  ┌────────────┐
              │helm deploy │  │helm deploy │  │helm deploy │
              │   dev      │  │  staging   │  │   live     │
              └────────────┘  └────────────┘  └────────────┘

          PR Pipeline:   build-pr → deploy-dev-pr → deploy-staging-pr
          Push Pipeline: build    → deploy-live
```

---

## GHA workflow openboa secrets

```
          Gitea Org Secrets
          ├── VAULT_ADDR
          ├── VAULT_ROLE_ID        → same for ALL workflows
          └── VAULT_SECRET_ID      → same for ALL workflows
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

## Openboa - microk8s - Go app secrets

```
          Running Pod (request time)
          ──────────────────────────
          K8s Secret (openbao-creds)         ← created manually via kubectl
          ├── VAULT_ADDR                         in each namespace (dev/staging/live)
          ├── VAULT_ROLE_ID
          └── VAULT_SECRET_ID
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
                            ├── client_cert    → LXD TLS auth
                            └── client_key     → LXD TLS auth
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
## Folder Structure

```
.
├── Gitea/
│   ├── docker-socket-runner.md
│   └── gitea-on-microk8s.md
├── LXD/
│   └── install-lxd.md
├── microk8s-vault-go on pod/
│   ├── install-microk8s.md
│   └── right-architecture.md
├── Minio/
│   └── install-minio.md
├── OpenBao/
│   ├── openbao-full-setup.md
│   └── openbao-installation-configuration.md
├── Secrets/
│   └── for-automation.md
└── README.md
```
