
#                   Architecture Diagrame
---

* [Infrastructure Archtecture - GHA - terraform - ansible](#infrastructure-archtecture---gha---terraform---ansible)
* [Infrastructure Archtecture  - GHA - kubernetes](#infrastructure-archtecture----gha---kubernetes)

* [Secrets Architecture - gha - org secrets - vault](#secrets-architecture---gha---org-secrets---vault)
* [Secrets Architecture - Go app - kubernetes - vault](#secrets-architecture---go-app---kubernetes---vault)

* [Promethius architecture](#promethius-architecture)

**Note** GHA /.gitea/workflow/x.yaml files pulled by runner is the center of all. runs everything in parallel
--- 

## Infrastructure Archtecture - GHA - terraform - ansible

```

                                                       ┌─────────────────────────┐ 
                                                       │   Gitea GHA workflow    │ 
                                                       |   (Actions triggers)    |
                                                       └─────────────────────────┘  
                                                                    │  
                                                                    │  
┌──────────────────────────────────────────────────────────┐        │       ┌─────────────────────────────────────────────┐
│                  Shared Gitea Repos (Infra org)          │        │       │           Gitea (gitea.local)               │  
│                                                          │        │       │   Repo with terrafrom, ansible, workflow    │
│  reusable-workflows-vault   ← Default  (OpenBao secrets) │        │       └───────┬─────────────────────────────────────┘
│  reusable-workflows         ← Alternate (org secrets)    │        │               |  
│  reusable-modules           ← Terraform lxd-vm module    │        │               |
│  reusable-ansible-galaxy-*  ← Ansible Galaxy roles       │        │               |              
└──────────────────────────────────────────────────┬───────┘        │               |
                                                   |                │               |
                                                   |                │               |        
                                              calls|                │               | calls                     
                                                   |                │               |    
                                                   |                ▼               |
                                        ┌──────────────────────────────────────────────────────┐                               
                                        │     Gitea Act Runner (Docker-based, inside LXD VM)   |
                                        |                                                      | 
                                        |  terrafrom ini/plan/apply           ansible-playbook | 
                                        └──────────┬─────────────────────────────┬─────────────┘                               
                                                   |                             |    
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

## Infrastructure Archtecture  - GHA - kubernetes

```
   
┌─────────────────────────┐ 
│   Gitea GHA workflow    │ 
|   (Actions triggers)    |
└─────────────────────────┘
        |
        |    
        |                                Gitea Repo
        |                   +-----------------------------------+
        |                   | Source Code + Helm Chart Folder   |
        |                   | Chart.yaml, values.yaml, templates|
        |                   | README.md                         |
        |                   +-----------------------------------+
        |                        |                     |        
        |                        |                     |                    
        |                        |                     | 
        |                        |                     |                                  
        |              +----------------------------------------------+         
        |              |                 Gite runner                  |  
        +--------------|                                              |              
                       | helm install/upgrade       Docker build push |-----------------+    
                       +----------------------------------------------+                 |   
                                        │                                               |                 
                                        |                                          +-----------------------+
                                        |                                          |Local registry / Harbor|
                                        |                                          +-----------------------+
                                        ▼                                                   |         
                +------------------------------------------------------------+              |
                |  Kubernetes Controllers (inside MicroK8s) creates pod      |              |   
                | - Deployment Controller ensures desired pods               |              |
                |  - Service Controller manages networking                   |              |  
                |  - Ingress Controller manages routing                      |              |  
                |  - Reconciliation loop keeps actual state = desired state  |              |   
                |- Kubelet on the node is the one that pulls the image       |              |
                |  (first time or when missing) and starts                   |              |
                +------------------------------------------------------------+              |
                                        |                                                   | 
                                        |                                                   |  
                                +----------------------+                                    |   
                                | Helm Metadata        |                                    |
                                | (Secrets/ConfigMaps) |                                    |
                                | - Release name       |                                    |   
                                | - Chart version      |                                    | 
                                | - Values used        |                                    |
                                | - History snapshot   |                                    |
                                +----------------------+                                    |
                                        │                                                   |
                                        ▼                                                   |
                                +----------------------+                                    |
                                | Kubernetes Objects   |                                    |
                                | (stored in etcd)     |                                    |    
                                | - Deployment (spec   |                                    |    
                                |   references registry|                                    |    
                                |   image)             |                                    | 
                                | - Service            |                                    |        
                                | - Ingress            |                                    | 
                                | - ConfigMaps/Secrets |                                    | 
                                +----------------------+                                    | 
                                        │                                                   | 
                                        ▼                                                   |
                                +--------------------------+                                |
                                | Pods (runtime)           |                                |    
                                |--------------------------|                                |
                                |- Kubelet Pull images from|                                |
                                |   MicroK8s registry      |   <----Kubelet on each node----+
                                | - Ephemeral, auto-       |        pulls images and starts 
                                |   recreated by K8s       |        containers
                                +--------------------------+

```
---

## Secrets Architecture - gha - org secrets - vault

```
        Gitea Org Secrets        - added manually via org/settings/actions/secrets
        ├── VAULT_ADDR
        ├── VAULT_ROLE_ID        → same for ALL workflows
        └── VAULT_SECRET_ID      → same for ALL workflows
                │
                ▼
                OpenBao
                (AppRole login)
                │
                ├── homelab/data/lxd        → LXD TLS cert/key     → GHA talks to LXD
                ├── homelab/data/minio      → MinIO creds          → Terraform state backend
                ├── homelab/data/ansible    → Ansible secrets      → Ansible playbooks
                └── homelab/data/microk8s   → kubeconfig           → helm deploy to MicroK8s

```
---

## Secrets Architecture - Go app - kubernetes - vault 

```
        Running Pod (request time)
        ──────────────────────────
        K8s Secret (openbao-creds)             ← created manually via kubectl create samespace and create secrets
        ├── VAULT_ADDR                             in each namespace (dev/staging/live)
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

## Promethius architecture

---

```
                                [ USER / ADMIN ]
                                        |
                         1. Apply YAML (ServiceMonitors, Rules) <----------------- Via gitea repo
                                        |
                                        v
[--------------------------- KUBERNETES API SERVER ---------------------------------]
                                        |
                                        | (Watches for changes)
                                        v
                            [ PROMETHEUS OPERATOR POD ]
                            "The Manager / Foreman"
                                        |
           -------------------------------------------------------------
           | (Configures)               | (Configures)                 | (Configures)
           v                            v                              v
   [ PROMETHEUS POD ] <---------- [ ALERTMANAGER ]                [ GRAFANA POD ]
   "The Database"      (Alerts)    "The Notifier"                 "The Visualizer"
           |                                                           |
           | 2. PULL / SCRAPE                                          | 3. QUERY
           | (Every 15-30s)                                            | (On Demand)
           |                                                           |
           |                                                           v
           |                                                     (User Web Browser)
           |                                                     "lxd-dashboard.local"
|          | 
           |
           |
           |                           
           |---> [ NODE EXPORTER ] ----> (Hardware/OS Metrics)
           |
           |---> [ YOUR GO APP ] 
                    |
                    |-- (:8080/metrics) <--- (Exposed via Go Client Library) (API for Prometheus to pull)
                    |
                    |-- [ INTERNAL APP LOGIC ]
                            |
                            |-- (LXD SDK) ----> [ LXD SERVER ]
                            |-- (Vault SDK) ---> [ OPENBAO ]
```