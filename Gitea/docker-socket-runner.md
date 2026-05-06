# Docker Socket Mode — Gitea Act Runner on Ubuntu 20.04

> Runner in Docker using Host Docker, registered with Gitea.

---

## Table of Contents

- [Step 1 — Obtain the Registration Token](#step-1--obtain-the-registration-token)
- [Step 2 — Prepare the Ubuntu Host](#step-2--prepare-the-ubuntu-host)
- [Step 3 — Create the Runner Workspace](#step-3--create-the-runner-workspace)
- [Step 4 — docker-compose.yml](#step-4--docker-composeyml)
- [Step 5 — Deploy the Runner](#step-5--deploy-the-runner)
- [Running Multiple Runners](#running-multiple-runners)

---

## Step 1 — Obtain the Registration Token

You must generate a token from your Gitea instance before configuring the host server.

1. Log in to your Gitea web interface.
2. Navigate to **Site Administration → Actions → Runners** (for a global runner), or to the settings of a specific repository/organization.
3. Click **Create new Runner**.
4. Copy the **Registration Token** and keep your **Gitea Instance URL** handy.

---

## Step 2 — Prepare the Ubuntu Host

Ensure the host system has the necessary dependencies installed and running.

```bash
# Update the package index
sudo apt update

# Install Docker and Docker Compose plugin
sudo apt install -y docker.io docker-compose-v2

# Enable Docker to start on boot and start the service immediately
sudo systemctl enable --now docker
```

---

## Step 3 — Create the Runner Workspace

Establish an isolated directory to hold the configuration and persistent data for the runner container.

```bash
mkdir -p ~/gitea-runner/data
cd ~/gitea-runner
```

---

## Step 4 — docker-compose.yml

Create the compose file:

```bash
vi docker-compose.yml
```

Paste the following:

```yaml
services:
  gitea-runner:
    image: gitea/act_runner:latest
    container_name: gitea-runner
    restart: always
    # Clears the work directory on start, then launches the runner
    command: >
      sh -c "rm -rf /data/_work/* && /sbin/tini -- /opt/bin/gitea-runner daemon"
    environment:
      # The reachable URL of your Gitea instance
      GITEA_INSTANCE_URL: "http://gitea.local"
      # The token generated in Step 1
      GITEA_RUNNER_REGISTRATION_TOKEN: "ZBD1mnh57Fxm3apuTWXFZelOqJ3vsJe3sJdVO667"
      # The display name for the Gitea UI
      GITEA_RUNNER_NAME: "ubuntu-docker-runner"
      # Mapping the 'ubuntu-latest' label to the official Gitea runner image
      GITEA_RUNNER_LABELS: "ubuntu-latest:docker://gitea/runner-images:ubuntu-latest"
    volumes:
      # Persist configuration and runner state
      - ./data:/data
      # Enable Docker Socket Mode (Docker-out-of-Docker)
      - /var/run/docker.sock:/var/run/docker.sock
```

---

## Step 5 — Deploy the Runner

Start the container. The `act_runner` daemon will automatically read the environment variables, contact your Gitea server, register itself, and apply the labels.

```bash
sudo docker compose up -d
```

---

## Running Multiple Runners

Create multiple folders with separate tokens from Gitea for each:

```
/gitea-runner1
/gitea-runner2
```

Get a different registration token from Gitea for each runner instance.
