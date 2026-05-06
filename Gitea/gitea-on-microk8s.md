# Gitea on MicroK8s

> For GHA + MicroK8s + Ansible/Terraform use cases. Gitea is lightweight, runs great on MicroK8s, and its Actions are GitHub Actions-compatible — so your workflows are portable.

---

## Table of Contents

- [1. Enable Required MicroK8s Addons](#1-enable-required-microk8s-addons)
- [2. Add Gitea Helm Chart](#2-add-gitea-helm-chart)
- [3. Create values.yaml](#3-create-valuesyaml)
- [4. Install Gitea](#4-install-gitea)
- [5. Check Deployment](#5-check-deployment)
- [6. Access Gitea](#6-access-gitea)

---

## 1. Enable Required MicroK8s Addons

```bash
microk8s enable dns storage ingress helm3
```

---

## 2. Add Gitea Helm Chart

```bash
microk8s helm3 repo add gitea-charts https://dl.gitea.com/charts/
microk8s helm3 repo update
```

---

## 3. Create values.yaml

```yaml
# gitea-values.yaml
gitea:
  admin:
    username: admin
    password: changeme123
    email: admin@local.dev

service:
  http:
    type: NodePort
    port: 3000

persistence:
  enabled: true
  size: 10Gi

actions:
  enabled: true   # Enables Gitea Actions (GHA-compatible)

postgresql-ha:
  enabled: false

postgresql:
  enabled: true
  primary:
    persistence:
      size: 5Gi
```

---

## 4. Install Gitea

```bash
microk8s helm3 install gitea gitea-charts/gitea \
  --namespace gitea \
  --create-namespace \
  -f gitea-values.yaml
```

---

## 5. Check Deployment

```bash
microk8s kubectl get pods -n gitea
microk8s kubectl get svc -n gitea
```

---

## 6. Access Gitea

```bash
# Get the NodePort
microk8s kubectl get svc gitea-http -n gitea

# Access via browser
http://<your-ubuntu-ip>:<nodeport>
```
