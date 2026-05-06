# Install MinIO on a VM (Linux)

---

## Table of Contents

- [1. Download MinIO Binary](#1-download-minio-binary)
- [Option A — Install as systemd Service (Recommended)](#option-a--install-as-systemd-service-recommended)
  - [1. Create Environment File](#1-create-environment-file)
  - [2. Create systemd Unit](#2-create-systemd-unit)
  - [3. Create User, Data Dir, and Start Service](#3-create-user-data-dir-and-start-service)
  - [4. Verify Listening](#4-verify-listening)

---

## 1. Download MinIO Binary

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
sudo mv minio /usr/local/bin/
```

---

## Option A — Install as systemd Service (Recommended)

### 1. Create Environment File

Create `/etc/default/minio`:

```ini
# /etc/default/minio
MINIO_VOLUMES="/var/minio/data"
MINIO_OPTS="--address :9000 --console-address :9001"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="minioadmin"
```

### 2. Create systemd Unit

Create `/etc/systemd/system/minio.service`:

```ini
[Unit]
Description=MinIO
Documentation=https://min.io
Wants=network-online.target
After=network-online.target

[Service]
User=minio
Group=minio
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 3. Create User, Data Dir, and Start Service

```bash
sudo useradd -r -s /sbin/nologin minio
sudo mkdir -p /var/minio/data
sudo chown -R minio:minio /var/minio

# Ensure /usr/local/bin/minio exists first
sudo systemctl daemon-reload
sudo systemctl enable --now minio
sudo journalctl -u minio -f
```

### 4. Verify Listening

```bash
ss -ltnp | grep :9000
curl -v http://127.0.0.1:9000/
```
