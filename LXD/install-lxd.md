# Install LXD (Snap-based)

---

## Table of Contents

- [1. Install LXD](#1-install-lxd)
- [2. Install the Web UI for LXD](#2-install-the-web-ui-for-lxd)
- [3. Access the Web UI](#3-access-the-web-ui)
- [4. Generate and Add Trust Password](#4-generate-and-add-trust-password)
- [5. Test by Launching a Container](#5-test-by-launching-a-container)
- [Optional — Enable Remote Access](#optional--enable-remote-access)

---

## 1. Install LXD

```bash
sudo apt update
sudo apt install snapd -y
sudo snap install lxd
```

Add your user to the LXD group:

```bash
sudo usermod -aG lxd $USER
newgrp lxd
```

Initialize LXD:

```bash
lxd init
```

> For most setups:
> - **Storage:** default (`dir` or `zfs` for advanced)
> - **Network:** create a bridge (recommended)
> - Accept defaults if unsure

---

## 2. Install the Web UI for LXD

The best modern option is the **official LXD UI**:

```bash
sudo snap install lxd-ui
lxd-ui enable
lxd-ui status
```

---

## 3. Access the Web UI

By default, it runs on:

```
https://<your-server-ip>:8443
```

To get your IP:

```bash
ip a
```

---

## 4. Generate and Add Trust Password

Set a trust password so the UI can connect:

```bash
lxc config set core.trust_password yourpassword
```

Then open the UI and:

1. Enter server address: `https://localhost:8443` (or your IP)
2. Enter the trust password

---

## 5. Test by Launching a Container

```bash
lxc launch images:ubuntu/22.04 my-container
lxc list
```

You should see the container inside the web UI instantly.

---

## Optional — Enable Remote Access

If accessing from another machine:

```bash
lxc config set core.https_address :8443
sudo ufw allow 8443/tcp
```
