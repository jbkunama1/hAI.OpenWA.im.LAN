<div align="center">

<img src="https://raw.githubusercontent.com/jbkunama1/hAI.OpenWA.im.LAN/main/logo_OPENWA.LAN.png" alt="hAI OpenWA im LAN Logo" width="400"/>

# 🤖 hAI · OpenWA im LAN
### *WhatsApp API Gateway — lokal, ohne Cloud, ohne Traefik*

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

> 💡 **Dieser Guide** löst alle bekannten Probleme beim Betrieb von OpenWA
> in einem lokalen Netzwerk — ohne Traefik, ohne Cloudflare-Timeout, ohne Dependency-Chaos.

</div>

---

## 📋 Inhaltsverzeichnis

- [🤔 Warum dieser Guide?](#-warum-dieser-guide)
- [✅ Voraussetzungen](#-voraussetzungen)
- [🔨 Schritt 1 – Dashboard lokal bauen](#-schritt-1--dashboard-lokal-bauen)
- [🐳 Schritt 2 – Portainer Stack](#-schritt-2--portainer-stack)
- [🧪 Schritt 3 – API testen](#-schritt-3--api-testen)
- [🖥️ Schritt 4 – Dashboard öffnen](#-schritt-4--dashboard-öffnen)
- [🔄 Auto-Update per Cron](#-auto-update-per-cron)
- [🐛 Bekannte Probleme & Lösungen](#-bekannte-probleme--lösungen)
- [🔌 Ports](#-ports)

---

## 🤔 Warum dieser Guide?

Das offizielle [OpenWA-Repo](https://github.com/rmyndharis/OpenWA) ist für Production mit **Traefik** ausgelegt.
Wer OpenWA aber auf einem **Heimserver, NAS oder Pi** ohne Traefik betreiben will, läuft direkt in mehrere Fallstricke:

| ⚠️ Problem | 🔍 Ursache | ✅ Lösung hier |
|---|---|---|
| `host not found in upstream "openwa"` | nginx.conf erwartet Traefik-Netzwerk | Standalone NGINX, kein Upstream-Proxy |
| `npm error ERESOLVE` | `vite@8` inkompatibel mit `@vitejs/plugin-react@5` | Node 20 + `--legacy-peer-deps` |
| ⏱️ Error 524 / Timeout | Build dauert >100s, Proxy bricht ab | Build per SSH — nicht in Portainer |
| `401 Invalid API key` | `X-API-Key`-Header fehlt oder Key falsch | Richtige Env-Variable + korrekter Header |
| NGINX-Standardseite | Falsches Dockerfile verwendet | Lokales Image `openwa-dashboard:local` nutzen |

---

## ✅ Voraussetzungen

```
🖥️  Server mit Docker & Docker Compose
📦  Portainer (optional — Stack-YAML läuft auch direkt)
🌐  Externes Docker-Netzwerk
🔑  SSH-Zugang zum Server
```

Netzwerk anlegen (falls noch nicht vorhanden):

```bash
docker network create highfishNetwork
```

---

## 🔨 Schritt 1 – Dashboard lokal bauen

> ⚠️ **Den Build NICHT über Portainer starten!**
> Der Prozess dauert länger als das Proxy-Timeout (~100 Sekunden)
> und schlägt mit **Error 524** fehl.
> → Einmalig per SSH bauen, danach läuft alles automatisch.

```bash
# 📥 Repo klonen
cd /opt
git clone --depth=1 https://github.com/rmyndharis/OpenWA.git openwa-src
cd openwa-src/dashboard

# 🔧 Bekannte Dependency-Konflikte fixen
# Problem: vite@8 ist inkompatibel mit @vitejs/plugin-react@5.1.4
sed -i 's/FROM node:.*/FROM node:20-alpine AS builder/' Dockerfile
sed -i 's/RUN npm ci/RUN npm install --legacy-peer-deps/' Dockerfile

# 🏗️ Dashboard-Image bauen
# YOUR_SERVER_IP = IP die dein Browser nutzt (z.B. 192.168.178.10)
docker build \
  --build-arg VITE_API_URL=http://YOUR_SERVER_IP:2785 \
  -t openwa-dashboard:local \
  .

# ✅ Prüfen ob Image da ist
docker images | grep openwa-dashboard
```

**Erwartete Ausgabe:**
```
REPOSITORY         TAG     IMAGE ID       CREATED         SIZE
openwa-dashboard   local   abc123def456   2 minutes ago   ~45MB
```

---

## 🐳 Schritt 2 – Portainer Stack

Neuen Stack in Portainer anlegen → **Web Editor** → YAML einfügen:

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
      # 🗄️ Datenbank
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
      # ⚡ Redis (deaktiviert)
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
      # 🔑 API Key — sicheren Wert setzen!
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
  # 🖥️ Dashboard (lokal gebaut — kein Build in Portainer)
  # ═══════════════════════════════════════
  dashboard:
    image: openwa-dashboard:local   # ← Schritt 1 muss vorher ausgeführt werden!
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

> 🔑 **API_MASTER_KEY** — einen zufälligen, sicheren Wert generieren:
> ```bash
> openssl rand -hex 32
> ```
> Diesen Key beim Dashboard-Login eingeben und als `X-API-Key`-Header in allen API-Requests verwenden.

---

## 🧪 Schritt 3 – API testen

```bash
# 🟢 Health Check (öffentlich — kein Key erforderlich)
curl -i http://YOUR_SERVER_IP:2785/api/health

# 🔐 Authentifizierter Endpoint (Key erforderlich)
curl -i http://YOUR_SERVER_IP:2785/api/health/detailed \
  -H "X-API-Key: YOUR_SECURE_API_KEY_HERE"
```

**✅ Erwartete Antwort (200 OK):**
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

**❌ Bei `401 Invalid API key`** — Key im Container prüfen:
```bash
docker exec -it openwa-api env | grep API_MASTER_KEY
# Ausgabe muss exakt dem gesetzten Key entsprechen
```

---

## 🖥️ Schritt 4 – Dashboard öffnen

```
🌐  Browser:   http://YOUR_SERVER_IP:8085
🔗  API-URL:   http://YOUR_SERVER_IP:2785
🔑  API-Key:   YOUR_SECURE_API_KEY_HERE
```

---

## 🔄 Auto-Update per Cron

Das Skript prüft täglich ob sich etwas im `dashboard/`-Ordner geändert hat.
Nur dann wird neu gebaut. Alte Images werden automatisch aufgeräumt — es bleiben immer die **2 neuesten** erhalten.

### Skript anlegen

```bash
cat > /opt/openwa-update.sh << 'EOF'
#!/bin/bash
set -e

REPO_DIR="/opt/openwa-src"
IMAGE_NAME="openwa-dashboard"
VITE_API_URL="http://YOUR_SERVER_IP:2785"
LOG_PREFIX="[openwa-update $(date '+%Y-%m-%d %H:%M:%S')]"

echo "$LOG_PREFIX 🚀 Start"

cd "$REPO_DIR"
git fetch origin main
CHANGES=$(git diff HEAD origin/main --name-only | grep "^dashboard/" || true)
git pull origin main

# 🔧 Dockerfile-Fixes nach jedem git pull neu anwenden
sed -i 's/FROM node:.*/FROM node:20-alpine AS builder/' dashboard/Dockerfile
sed -i 's/RUN npm ci/RUN npm install --legacy-peer-deps/' dashboard/Dockerfile

if [ -z "$CHANGES" ]; then
  echo "$LOG_PREFIX ✅ Keine Dashboard-Änderungen, überspringe Build."
  exit 0
fi

echo "$LOG_PREFIX 🔨 Änderungen erkannt, baue neu..."

TIMESTAMP=$(date '+%Y%m%d%H%M%S')
docker build \
  --build-arg VITE_API_URL="$VITE_API_URL" \
  -t "$IMAGE_NAME:$TIMESTAMP" \
  -t "$IMAGE_NAME:latest" \
  "$REPO_DIR/dashboard"

docker restart openwa-dashboard
echo "$LOG_PREFIX ♻️  Container neu gestartet."

# 🧹 Alte Images aufräumen: nur 2 neueste behalten
echo "$LOG_PREFIX 🗑️  Räume alte Images auf..."
docker images "$IMAGE_NAME" --format "{{.Tag}}\t{{.ID}}" \
  | grep -v "latest" \
  | sort -r \
  | tail -n +3 \
  | awk '{print $2}' \
  | xargs -r docker rmi

echo "$LOG_PREFIX ✅ Fertig. Aktuelle Images:"
docker images "$IMAGE_NAME"
EOF

chmod +x /opt/openwa-update.sh
```

### Cron einrichten

```bash
crontab -e
# Zeile einfügen — läuft täglich um 3:00 Uhr:
0 3 * * * /opt/openwa-update.sh >> /var/log/openwa-update.log 2>&1
```

### Manuell testen & Logs prüfen

```bash
# Einmalig manuell ausführen
/opt/openwa-update.sh

# Logs live verfolgen
tail -f /var/log/openwa-update.log
```

---

## 🐛 Bekannte Probleme & Lösungen

<details>
<summary>❌ <b>host not found in upstream "openwa"</b></summary>

**Ursache:** Das Dashboard-`Dockerfile.traefik` nutzt einen nginx-Upstream der nur im Traefik-Netzwerk auflösbar ist.

**Lösung:** Sicherstellen dass im Stack `image: openwa-dashboard:local` steht, **nicht** ein `build:`-Block der auf das GitHub-Repo zeigt. Das lokal gebaute Image enthält die richtige `nginx.conf` ohne Upstream-Proxy.
</details>

<details>
<summary>❌ <b>npm error ERESOLVE / exit code 1</b></summary>

**Ursache:** `vite@8.x` (im Repo) ist inkompatibel mit `@vitejs/plugin-react@5.1.4` (unterstützt nur bis vite@7).

**Lösung:** Die beiden `sed`-Befehle aus Schritt 1 ausführen — Node 20 pinnen und `npm install --legacy-peer-deps` setzen.
</details>

<details>
<summary>❌ <b>Error 524 / Timeout beim Portainer-Deploy</b></summary>

**Ursache:** Cloudflare und andere Proxies trennen HTTP-Verbindungen nach ~100 Sekunden. Der npm-Build dauert länger.

**Lösung:** Dashboard **nie** über Portainer bauen. Immer per SSH (Schritt 1). Danach nur noch `image: openwa-dashboard:local` im Stack.
</details>

<details>
<summary>❌ <b>401 Invalid API key</b></summary>

**Ursache:** `API_MASTER_KEY` ist leer, falsch geschrieben, oder der falsche Header wird verwendet.

**Lösung:**
```bash
# Key im laufenden Container prüfen
docker exec -it openwa-api env | grep API_MASTER_KEY

# Korrekter Header in curl:
curl -H "X-API-Key: DEIN_KEY" http://IP:2785/api/health/detailed
```
</details>

<details>
<summary>❌ <b>NGINX zeigt nur Standardseite</b></summary>

**Ursache:** Falsches Image oder Build nicht ausgeführt.

**Lösung:** Schritt 1 erneut ausführen, dann `docker images | grep openwa-dashboard` prüfen. Image muss vorhanden sein bevor der Stack deployed wird.
</details>

---

## 🔌 Ports

| Service | Port | URL | Beschreibung |
|---|---|---|---|
| 🤖 API | `2785` | `http://IP:2785/api` | REST API |
| 📖 Swagger | `2785` | `http://IP:2785/api/docs` | Interaktive API-Doku |
| 🖥️ Dashboard | `8085` | `http://IP:8085` | Web-UI (Standalone) |

---

## 🔗 Links

[![OpenWA Upstream](https://img.shields.io/badge/OpenWA-Upstream%20Repo-6366f1?style=flat-square&logo=github)](https://github.com/rmyndharis/OpenWA)
[![Issues](https://img.shields.io/badge/OpenWA-Issues-ef4444?style=flat-square&logo=github)](https://github.com/rmyndharis/OpenWA/issues)
[![Portainer Docs](https://img.shields.io/badge/Portainer-Dokumentation-13BEF9?style=flat-square&logo=portainer)](https://docs.portainer.io)
[![Docker Docs](https://img.shields.io/badge/Docker-Dokumentation-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com)

---

<div align="center">

**hAI · OpenWA im LAN** — Self-hosted, Open Source, kein Cloud-Zwang 🏠

<sub>Erstellt von <a href="https://github.com/jbkunama1">therealteacher</a> · Karlsruhe 🇩🇪</sub>

</div>
