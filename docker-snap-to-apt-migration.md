# Docker Snap → apt Migration & VPS Footprint Optimization

> **Target:** Ubuntu Server VPS · 20 GB volume · 2 GB RAM · AI agents via Docker Compose

---

## The Problem

`/var/snap` consuming **11 GB** is caused by Docker being installed via snap. The Docker snap bundles containerd, runc, CLI, and image cache entirely inside `/var/snap/docker/`.

```bash
snap list
# Name    Version   Rev    Tracking       Publisher   Notes
# core24  20260410  1643   latest/stable  canonical✓  base
# docker  29.3.1    3505   latest/stable  canonical✓  -
# snapd   2.75.2    26865  latest/stable  canonical✓  snapd
```

---

## Approach Comparison: `uv .venv` vs Docker Compose

| Factor | `uv .venv` | Docker Compose |
|---|---|---|
| **Disk overhead** | ~50–200 MB per venv | ~200–800 MB per image (base layer) |
| **RAM overhead** | Near zero (no daemon) | ~50–150 MB (dockerd + containerd always running) |
| **Shared dependencies** | ✅ Shared across venvs via uv cache | ❌ Each image bundles its own copies |
| **Python runtime** | System Python, shared | Bundled per image (unless slim/alpine) |
| **Startup time** | Near instant | 1–5s container startup |
| **Process isolation** | ❌ No namespace isolation | ✅ Full namespace isolation |
| **Networking** | Direct host networking | Virtual bridge (slight overhead) |
| **Orchestration** | Manual / systemd | Built-in via compose |

### Verdict: `uv .venv` wins on a 20 GB / 2 GB RAM VPS

- **No daemon tax** — Docker always runs `dockerd` + `containerd` (~100–200 MB RAM before first container)
- **uv global cache deduplicates packages** — `httpx`, `langchain`, etc. stored once across all agents
- **Heavy AI deps (torch, transformers)** — paid per image with Docker; shared with uv
- **`python:3.12-slim` is still ~150 MB per image** — 3–5 agents = 450–750 MB in base layers alone

---

## Fix Option A: Stay with Docker but Remove the Snap

If you need Docker (for isolation, existing compose workflows, etc.), replace snap-Docker with apt-Docker.

### Step 1 — Audit current state

```bash
docker ps -a
docker images
docker compose ls

# Locate your compose files
find / -name "docker-compose.yml" 2>/dev/null
```

### Step 2 — Stop all containers gracefully

```bash
# Stop per project
docker compose down

# Or stop all at once
docker stop $(docker ps -q) 2>/dev/null
```

### Step 3 — Remove Docker snap + core24 + snapd

```bash
sudo snap remove --purge docker
sudo snap remove --purge core24
sudo snap remove --purge snapd

# Prevent snapd from being reinstalled by apt
sudo apt-mark hold snapd
sudo apt purge snapd -y

# Clean up all snap remnants
sudo rm -rf /var/snap /var/lib/snapd /snap ~/snap
```

> ⚠️ All Docker images, volumes, and containers stored under `/var/snap/docker/` will be deleted.
> Your compose YAML files are safe — images will be re-pulled from Docker Hub on next startup.

### Step 4 — Install Docker Engine via official apt repo

```bash
# Install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### Step 5 — Enable and configure

```bash
sudo systemctl enable --now docker
docker version
docker compose version

# Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker
```

### Step 6 — Restore your agents

```bash
cd /opt/your-agent-project
docker compose up -d
# Images will be re-pulled from Docker Hub automatically
```

### What you gain by switching Docker snap → apt

| | Docker via Snap | Docker via apt |
|---|---|---|
| **Storage location** | `/var/snap/docker/` | `/var/lib/docker/` |
| **`/var/snap` usage** | ~11 GB | **~0 GB** |
| **Update mechanism** | snapd daemon | `apt upgrade` |
| **systemd integration** | Indirect | ✅ Native |
| **Startup** | Slower (snap mount overhead) | Faster |
| **Docker socket path** | `/var/run/docker.sock` | `/var/run/docker.sock` (same) |

---

## Fix Option B: Drop Docker Entirely — Use `uv` + systemd

Best choice for maximum footprint reduction on a resource-constrained VPS.

### Project structure

```
/opt/agents/
├── agent1/
│   ├── .venv/           ← uv-managed virtualenv
│   ├── pyproject.toml
│   └── main.py
├── agent2/
│   ├── .venv/
│   ├── pyproject.toml
│   └── main.py
└── shared/              ← optional common utilities
```

### Install uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
# Single binary, ~10 MB
# Package cache lives at ~/.cache/uv — shared across all venvs
```

### Setup per agent

```bash
cd /opt/agents/agent1
uv init --no-workspace
uv add langchain httpx openai   # add your dependencies
uv run main.py                  # run directly
```

### Manage agents with systemd

```ini
# /etc/systemd/system/agent1.service

[Unit]
Description=AI Agent 1
After=network.target

[Service]
Type=simple
User=agentuser
WorkingDirectory=/opt/agents/agent1
ExecStart=/opt/agents/agent1/.venv/bin/python main.py
Restart=on-failure
RestartSec=5
Environment=PYTHONUNBUFFERED=1
EnvironmentFile=/opt/agents/agent1/.env

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now agent1
sudo systemctl status agent1
journalctl -fu agent1    # live logs
```

---

## Quick Wins — Apply Regardless of Approach

```bash
# 1. Remove snap (recover ~8–11 GB immediately)
sudo snap remove --purge docker
sudo snap remove --purge core24
sudo snap remove --purge snapd
sudo apt purge snapd -y
sudo apt-mark hold snapd
sudo rm -rf /var/snap /var/lib/snapd /snap ~/snap

# 2. Clean apt cache
sudo apt autoremove -y && sudo apt clean

# 3. If keeping Docker, prune unused data
docker system prune -af --volumes

# 4. Verify recovered space
df -h
```

---

## Expected Disk Recovery

| Action | Estimated Recovery |
|---|---|
| Remove Docker snap + core24 + snapd | **~8–11 GB** |
| `apt autoremove` + `apt clean` | ~200–500 MB |
| `docker system prune` (if staying on Docker) | Varies (image-dependent) |
| Switch to `uv` (no Docker images at all) | Saves ~500 MB–2 GB ongoing |

After removing snap and switching to `uv` + systemd, expected disk usage drops from **~18 GB → ~5–8 GB**, with noticeably more free RAM due to no dockerd/containerd daemon overhead.

---

*Ubuntu Server · QEMU/KVM compatible · Tested on Ubuntu 22.04 / 24.04 LTS*
