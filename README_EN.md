<div align="center">

<img src="https://raw.githubusercontent.com/jbkunama1/hAI.OpenWA.im.LAN/main/logo_OPENWA.LAN.png" alt="hAI OpenWA in the LAN Logo" width="400"/>

# 🤖 hAI · OpenWA in the LAN
### *WhatsApp API Gateway — self-hosted, no cloud, no Traefik*

<br/>

[![based on](https://img.shields.io/badge/based%20on-OpenWA%200.1.6-6366f1?style=for-the-badge&logo=github)](https://github.com/rmyndharis/OpenWA)
[![node](https://img.shields.io/badge/Node.js-20%20LTS-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![docker](https://img.shields.io/badge/Docker-ready-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![portainer](https://img.shields.io/badge/Portainer-Stack-13BEF9?style=for-the-badge&logo=portainer&logoColor=white)](https://portainer.io)
[![traefik](https://img.shields.io/badge/Traefik-not%20required-lightgrey?style=for-the-badge)]()
[![license](https://img.shields.io/badge/License-MIT-22c55e?style=for-the-badge)](LICENSE)

<br/>

[![Buy me a coffee](https://cdn.buymeacoffee.com/buttons/default-orange.png)](https://www.buymeacoffee.com/highfish)

<br/>

> 💡 **This guide** solves all known issues when running OpenWA
> in a local network — no Traefik, no Cloudflare timeout, no dependency chaos.

</div>

---

## 📋 Table of Contents

- [🤔 Why this guide?](#-why-this-guide)
- [✅ Prerequisites](#-prerequisites)
- [🔨 Step 1 – Build the dashboard locally](#-step-1--build-the-dashboard-locally)
- [🐳 Step 2 – Portainer Stack](#-step-2--portainer-stack)
- [🧪 Step 3 – Test the API](#-step-3--test-the-api)
- [🖥️ Step 4 – Open the dashboard](#-step-4--open-the-dashboard)
- [🔄 Auto-update via Cron](#-auto-update-via-cron)
- [🐛 Known Issues & Solutions](#-known-issues--solutions)
- [🔌 Ports](#-ports)

---

## 🤔 Why this guide?

The official [OpenWA repo](https://github.com/rmyndharis/OpenWA) is designed for production with **Traefik**.
Anyone running OpenWA on a **home server, NAS, or Pi** without Traefik will immediately hit several pitfalls:

| ⚠️ Problem | 🔍 Cause | ✅ Solution here |
|---|---|---|
| `host not found in upstream "openwa"` | nginx.conf expects a Traefik network | Standalone NGINX, no upstream proxy |
| `npm error ERESOLVE` | `vite@8` incompatible with `@vitejs/plugin-react@5` | Node 20 + `--legacy-peer-deps` |
| ⏱️ Error 524 / Timeout | Build takes >100s, proxy disconnects | Build via SSH — not in Portainer |
| `401 Invalid API key` | `X-API-Key` header missing or wrong | Correct env variable + correct header |
| NGINX default page | Wrong Dockerfile used | Use local image `openwa-dashboard:local` |

---

## ✅ Prerequisites

```
🖥️  Server with Docker & Docker Compose
📦  Portainer (optional — Stack YAML works standalone too)
🌐  External Docker network
🔑  SSH access to the server
```

Create the network (if not already present):

```bash
docker network create highfishNetwork
```

---

## 🔨 Step 1 – Build the dashboard locally

> ⚠️ **Do NOT trigger the build via Portainer!**
> The process takes longer than the proxy timeout (~100 seconds)
> and will fail with **Error 524**.
> → Build once via SSH — after that everything runs automatically.

```bash
# 📥 Clone the repo
cd /opt
git clone --depth=1 https://github.com/rmyndharis/OpenWA.git openwa-src
cd openwa-src/dashboard

# 🔧 Fix known dependency conflicts
# Problem: vite@8 is incompatible with @vitejs/plugin-react@5.1.4
sed -i 's/FROM node:.*/FROM node:20-alpine AS builder/' Dockerfile
sed -i 's/RUN npm ci/RUN npm install --legacy-peer-deps/' Dockerfile

# 🏗️ Build the dashboard image
# YOUR_SERVER_IP = IP your browser uses (e.g. YOURS)
docker build \
  --build-arg VITE_API_URL=http://YOUR_SERVER_IP:2785 \
  -t openwa-dashboard:local \
  .

# ✅ Verify the image exists
docker images | grep openwa-dashboard
```

**Expected output:**
```
REPOSITORY         TAG     IMAGE ID       CREATED         SIZE
openwa-dashboard   local   abc123def456   2 minutes ago   ~45MB
```

---

## 🐳 Step 2 – Portainer Stack

Create a new stack in Portainer → **Web Editor** → paste the YAML:

```yaml
version: "3.8"

services:

  # ═══════════════════════════════════════
  # 🤖 OpenWA API
  # ═══════════════════════════════════════
  openwa-api:
    build:
      context: https://github.com/rmyndharis/OpenWA.git
      dockerfile: Dockerfile
    container_name: openwa-api
    restart: unless-stopped
    ports:
      - "2785:2785"
    environment:
      NODE_ENV: production
      PORT: 2785
      LOG_LEVEL: info
      # 🗄️ Database
      DATABASE_TYPE: sqlite
      DATABASE_NAME: /app/data/openwa.sqlite
      # 💬 WhatsApp Engine
      ENGINE_TYPE: whatsapp-web.js
      SESSION_DATA_PATH: /app/data/sessions
      PUPPETEER_HEADLESS: "true"
      PUPPETEER_ARGS: --no-sandbox,--disable-setuid-sandbox,--disable-dev-shm-usage,--disable-gpu
      # 📁 Storage
      STORAGE_TYPE: local
      STORAGE_LOCAL_PATH: /app/data/media
      # ⚡ Redis (disabled)
      REDIS_ENABLED: "false"
      # 🪝 Webhook
      WEBHOOK_TIMEOUT: "10000"
      WEBHOOK_MAX_RETRIES: "3"
      WEBHOOK_RETRY_DELAY: "5000"
      # 🚦 Rate Limiting
      RATE_LIMIT_TTL: "60"
      RATE_LIMIT_MAX: "100"
      # 🔌 Plugins
      PLUGINS_ENABLED: "true"
      PLUGINS_DIR: /app/data/plugins
      # 🔑 API Key — set a secure value!
      API_MASTER_KEY: "YOUR_SECURE_API_KEY_HERE"
    volumes:
      - openwa-data:/app/data
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:2785/api/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - highfishNetwork

  # ═══════════════════════════════════════
  # 🖥️ Dashboard (built locally — no build in Portainer)
  # ═══════════════════════════════════════
  dashboard:
    image: openwa-dashboard:local   # ← Step 1 must be completed first!
    container_name: openwa-dashboard
    restart: unless-stopped
    ports:
      - "8085:80"
    depends_on:
      - openwa-api
    networks:
      - highfishNetwork

volumes:
  openwa-data:
    driver: local

networks:
  highfishNetwork:
    external: true
```

> 🔑 **API_MASTER_KEY** — generate a random, secure value:
> ```bash
> openssl rand -hex 32
> ```
> Use this key when logging into the dashboard and as the `X-API-Key` header in all API requests.

---

## 🧪 Step 3 – Test the API

```bash
# 🟢 Health check (public — no key required)
curl -i http://YOUR_SERVER_IP:2785/api/health

# 🔐 Authenticated endpoint (key required)
curl -i http://YOUR_SERVER_IP:2785/api/health/detailed \
  -H "X-API-Key: YOUR_SECURE_API_KEY_HERE"
```

**✅ Expected response (200 OK):**
```json
{
  "status": "ok",
  "version": "0.1.0",
  "uptime": 120,
  "checks": {
    "database": "ok"
  }
}
```

**❌ On `401 Invalid API key`** — check the key inside the container:
```bash
docker exec -it openwa-api env | grep API_MASTER_KEY
# Output must exactly match the key you set
```

---

## 🖥️ Step 4 – Open the dashboard

```
🌐  Browser:   http://YOUR_SERVER_IP:8085
🔗  API URL:   http://YOUR_SERVER_IP:2785
🔑  API Key:   YOUR_SECURE_API_KEY_HERE
```

---

## 🔄 Auto-update via Cron

The script checks daily whether anything in the `dashboard/` folder has changed.
A rebuild only happens when changes are detected. Old images are cleaned up automatically — only the **2 most recent** are kept.

### Create the script

```bash
cat > /opt/openwa-update.sh << 'EOF'
#!/bin/bash
set -e

REPO_DIR="/opt/openwa-src"
IMAGE_NAME="openwa-dashboard"
VITE_API_URL="http://YOUR_SERVER_IP:2785"
LOG_PREFIX="[openwa-update $(date '+%Y-%m-%d %H:%M:%S')]"

echo "$LOG_PREFIX 🚀 Starting"

cd "$REPO_DIR"
git fetch origin main
CHANGES=$(git diff HEAD origin/main --name-only | grep "^dashboard/" || true)
git pull origin main

# 🔧 Re-apply Dockerfile fixes after every git pull
sed -i 's/FROM node:.*/FROM node:20-alpine AS builder/' dashboard/Dockerfile
sed -i 's/RUN npm ci/RUN npm install --legacy-peer-deps/' dashboard/Dockerfile

if [ -z "$CHANGES" ]; then
  echo "$LOG_PREFIX ✅ No dashboard changes, skipping build."
  exit 0
fi

echo "$LOG_PREFIX 🔨 Changes detected, rebuilding..."

TIMESTAMP=$(date '+%Y%m%d%H%M%S')
docker build \
  --build-arg VITE_API_URL="$VITE_API_URL" \
  -t "$IMAGE_NAME:$TIMESTAMP" \
  -t "$IMAGE_NAME:latest" \
  "$REPO_DIR/dashboard"

docker restart openwa-dashboard
echo "$LOG_PREFIX ♻️  Container restarted."

# 🧹 Clean up old images: keep only the 2 most recent
echo "$LOG_PREFIX 🗑️  Cleaning up old images..."
docker images "$IMAGE_NAME" --format "{{.Tag}}\t{{.ID}}" \
  | grep -v "latest" \
  | sort -r \
  | tail -n +3 \
  | awk '{print $2}' \
  | xargs -r docker rmi

echo "$LOG_PREFIX ✅ Done. Current images:"
docker images "$IMAGE_NAME"
EOF

chmod +x /opt/openwa-update.sh
```

### Set up the cron job

```bash
crontab -e
# Add this line — runs daily at 3:00 AM:
0 3 * * * /opt/openwa-update.sh >> /var/log/openwa-update.log 2>&1
```

### Test manually & follow logs

```bash
# Run once manually
/opt/openwa-update.sh

# Follow logs live
tail -f /var/log/openwa-update.log
```

---

## 🐛 Known Issues & Solutions

<details>
<summary>❌ <b>host not found in upstream "openwa"</b></summary>

**Cause:** The dashboard `Dockerfile.traefik` uses an nginx upstream that can only be resolved inside a Traefik network.

**Fix:** Make sure the stack uses `image: openwa-dashboard:local`, **not** a `build:` block pointing to the GitHub repo. The locally built image contains the correct `nginx.conf` without an upstream proxy.
</details>

<details>
<summary>❌ <b>npm error ERESOLVE / exit code 1</b></summary>

**Cause:** `vite@8.x` (in the repo) is incompatible with `@vitejs/plugin-react@5.1.4` (which only supports up to vite@7).

**Fix:** Run the two `sed` commands from Step 1 — pin Node 20 and switch to `npm install --legacy-peer-deps`.
</details>

<details>
<summary>❌ <b>Error 524 / Timeout during Portainer deploy</b></summary>

**Cause:** Cloudflare and other proxies drop HTTP connections after ~100 seconds. The npm build takes longer.

**Fix:** **Never** build the dashboard through Portainer. Always use SSH (Step 1). Afterwards only use `image: openwa-dashboard:local` in the stack.
</details>

<details>
<summary>❌ <b>401 Invalid API key</b></summary>

**Cause:** `API_MASTER_KEY` is empty, misspelled, or the wrong header is used.

**Fix:**
```bash
# Check the key inside the running container
docker exec -it openwa-api env | grep API_MASTER_KEY

# Correct header in curl:
curl -H "X-API-Key: YOUR_KEY" http://IP:2785/api/health/detailed
```
</details>

<details>
<summary>❌ <b>NGINX only shows the default page</b></summary>

**Cause:** Wrong image used or build was not executed.

**Fix:** Re-run Step 1, then check `docker images | grep openwa-dashboard`. The image must exist before the stack is deployed.
</details>

---

## 🔌 Ports

| Service | Port | URL | Description |
|---|---|---|---|
| 🤖 API | `2785` | `http://IP:2785/api` | REST API |
| 📖 Swagger | `2785` | `http://IP:2785/api/docs` | Interactive API docs |
| 🖥️ Dashboard | `8085` | `http://IP:8085` | Web UI (standalone) |

---

## 🔗 Links

[![OpenWA Upstream](https://img.shields.io/badge/OpenWA-Upstream%20Repo-6366f1?style=flat-square&logo=github)](https://github.com/rmyndharis/OpenWA)
[![Issues](https://img.shields.io/badge/OpenWA-Issues-ef4444?style=flat-square&logo=github)](https://github.com/rmyndharis/OpenWA/issues)
[![Portainer Docs](https://img.shields.io/badge/Portainer-Documentation-13BEF9?style=flat-square&logo=portainer)](https://docs.portainer.io)
[![Docker Docs](https://img.shields.io/badge/Docker-Documentation-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com)

---

<div align="center">

**hAI · OpenWA in the LAN** — Self-hosted, open source, no cloud required 🏠

<sub>Created by <a href="https://github.com/jbkunama1">therealteacher</a> · Karlsruhe 🇩🇪</sub>

</div>
