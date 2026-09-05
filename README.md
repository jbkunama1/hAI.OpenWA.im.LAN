<div align="center">

<img src="https://raw.githubusercontent.com/jbkunama1/hAI.OpenWA.im.LAN/main/logo_OPENWA.LAN.png" alt="hAI OpenWA im LAN Logo" width="400"/>

# ðŸ¤– hAI Â· OpenWA im LAN
### *WhatsApp API Gateway â€” lokal, ohne Cloud, ohne Traefik*

<br/>

[![based on](https://img.shields.io/badge/based%20on-OpenWA%200.1.6-6366f1?style=for-the-badge&logo=github)](https://github.com/rmyndharis/OpenWA)
[![node](https://img.shields.io/badge/Node.js-20%20LTS-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![docker](https://img.shields.io/badge/Docker-ready-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![portainer](https://img.shields.io/badge/Portainer-Stack-13BEF9?style=for-the-badge&logo=portainer&logoColor=white)](https://portainer.io)
[![traefik](https://img.shields.io/badge/Traefik-not%20required-lightgrey?style=for-the-badge)](https://github.com/jbkunama1/hAI.OpenWA.im.LAN)
[![ghcr.io](https://img.shields.io/badge/ghcr.io-images%20available-2496ED?style=for-the-badge&logo=github&logoColor=white)](https://github.com/jbkunama1/hAI.OpenWA.im.LAN/pkgs/container/openwa-api)
[![license](https://img.shields.io/badge/License-MIT-22c55e?style=for-the-badge)](LICENSE)

<br/>

[![Buy me a coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://www.buymeacoffee.com/highfish)

<br/>

> ðŸ’¡ **Dieser Guide** lÃ¶st alle bekannten Probleme beim Betrieb von OpenWA
> in einem lokalen Netzwerk â€” ohne Traefik, ohne Cloudflare-Timeout, ohne Dependency-Chaos.

</div>

---

## ðŸ“‹ Inhaltsverzeichnis

- [ðŸ¤” Warum dieser Guide?](#-warum-dieser-guide)
- [âœ… Voraussetzungen](#-voraussetzungen)
- [ðŸ”¨ Schritt 1 â€“ Images lokal bauen](#-schritt-1--images-lokal-bauen)
- [ðŸ³ Schritt 2 â€“ Portainer Stack](#-schritt-2--portainer-stack)
- [⚡ Schnellstart mit ghcr.io (Empfohlen)](#-schnellstart-mit-ghcrio-empfohlen)
- [ðŸ§ª Schritt 3 â€“ API testen](#-schritt-3--api-testen)
- [ðŸ–¥ï¸ Schritt 4 â€“ Dashboard Ã¶ffnen](#-schritt-4--dashboard-Ã¶ffnen)
- [ðŸ”„ Auto-Update per Cron](#-auto-update-per-cron)
- [ðŸ› Bekannte Probleme & LÃ¶sungen](#-bekannte-probleme--lÃ¶sungen)
- [ðŸ”Œ Ports](#-ports)

---

## ðŸ¤” Warum dieser Guide?

Das offizielle [OpenWA-Repo](https://github.com/rmyndharis/OpenWA) ist fÃ¼r Production mit **Traefik** ausgelegt.
Wer OpenWA aber auf einem **Heimserver, NAS oder Pi** ohne Traefik betreiben will, lÃ¤uft direkt in mehrere Fallstricke:

| âš ï¸ Problem | ðŸ” Ursache | âœ… LÃ¶sung hier |
|---|---|---|
| `host not found in upstream "openwa"` | nginx.conf erwartet Traefik-Netzwerk | Standalone NGINX, kein Upstream-Proxy |
| `npm error ERESOLVE` | `vite@8` inkompatibel mit `@vitejs/plugin-react@5` | Node 20 + `--legacy-peer-deps` |
| â±ï¸ Error 524 / Timeout | Build dauert >100s, Proxy bricht ab | **Beide** Images per SSH bauen â€” nicht in Portainer |
| `401 Invalid API key` | `X-API-Key`-Header fehlt oder Key falsch | Richtige Env-Variable + korrekter Header |
| NGINX-Standardseite | Falsches Dockerfile verwendet | Lokale Images `openwa-api:local` + `openwa-dashboard:local` nutzen |

---

## âœ… Voraussetzungen

```
ðŸ–¥ï¸  Server mit Docker & Docker Compose
ðŸ“¦  Portainer (optional â€” Stack-YAML lÃ¤uft auch direkt)
ðŸŒ  Externes Docker-Netzwerk
ðŸ”‘  SSH-Zugang zum Server
ðŸŒ  Zwei Subdomains:
       Dashboard: YOURS
       API:       YOURS
```

Netzwerk anlegen (falls noch nicht vorhanden):

```bash
docker network create highfishNetwork
```

---


---

## ⚡ Schnellstart mit ghcr.io (Empfohlen)

Für die meisten Nutzer ist es am einfachsten, die bereits fertig gebauten Docker-Images aus der **GitHub Container Registry (ghcr.io)** zu verwenden. Dies erspart den lokalen Build-Prozess und läuft direkt.

### Images ziehen

```bash
docker pull ghcr.io/jbkunama1/openwa-api:latest
docker pull ghcr.io/jbkunama1/openwa-dashboard:latest
```

### Portainer Stack mit ghcr.io

Nutze die Datei `docker-compose.ghcr.yml` aus diesem Repository oder kopiere diesen YAML-Block direkt in deinen Portainer Stack. Ersetze `YOUR_SECURE_API_KEY_HERE` durch einen sicheren Key.
## ðŸ”¨ Schritt 1 â€“ Images lokal bauen

> âš ï¸ **Den Build NICHT Ã¼ber Portainer starten!**
> Der Prozess dauert lÃ¤nger als das Proxy-Timeout (~100 Sekunden)
> und schlÃ¤gt mit **Error 524** fehl.
> â†’ Einmalig per SSH bauen, danach lÃ¤uft alles automatisch.

### 1a â€“ Backend-API bauen

```bash
# ðŸ“¥ Repo klonen
cd /opt
git clone --depth=1 https://github.com/rmyndharis/OpenWA.git openwa-src

# ðŸ—ï¸ Backend-Image bauen (~3â€“5 Min, npm warn deprecated = normal, kein Fehler)
docker build \
  https://github.com/rmyndharis/OpenWA.git#main \
  -t openwa-api:local
```

### 1b â€“ Dashboard bauen

```bash
cd /opt/openwa-src/dashboard

# ðŸ”§ Bekannte Dependency-Konflikte fixen
# Problem: vite@8 ist inkompatibel mit @vitejs/plugin-react@5.1.4
sed -i 's/RUN npm ci/RUN npm ci --legacy-peer-deps/' Dockerfile

# ðŸ—ï¸ Dashboard-Image bauen
# VITE_API_URL zeigt auf die API-Subdomain
docker build \
  --build-arg VITE_API_URL=https://YOURS \
  -t openwa-dashboard:local \
  .
```

> ðŸ’¡ **VITE_API_URL** ist eine **Build-Zeit-Variable** â€” sie wird fest ins JS-Bundle eingebaut.
> Das Dashboard unter `YOURS` spricht die API unter `YOURS` an.

### âœ… PrÃ¼fen ob beide Images da sind

```bash
docker images | grep openwa
# Erwartete Ausgabe:
# openwa-api        local   abc123...   5 minutes ago   ~500MB
# openwa-dashboard  local   def456...   2 minutes ago   ~45MB
```

---

## ðŸ³ Schritt 2 â€“ Portainer Stack

Neuen Stack in Portainer anlegen â†’ **Web Editor** â†’ YAML einfÃ¼gen:

```yaml
version: "3.8"

services:

  # â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
  # ðŸ¤– OpenWA API (lokal gebaut)
  # â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
  openwa-api:
    image: openwa-api:local          # â† Schritt 1a muss vorher ausgefÃ¼hrt werden!
    container_name: openwa
    restart: unless-stopped
    ports:
      - "2785:2785"
    environment:
      NODE_ENV: production
      PORT: 2785
      LOG_LEVEL: info
      # ðŸ—„ï¸ Datenbank
      DATABASE_TYPE: sqlite
      DATABASE_NAME: /app/data/openwa.sqlite
      # ðŸ’¬ WhatsApp Engine
      ENGINE_TYPE: whatsapp-web.js
      SESSION_DATA_PATH: /app/data/sessions
      PUPPETEER_HEADLESS: "true"
      PUPPETEER_ARGS: --no-sandbox,--disable-setuid-sandbox,--disable-dev-shm-usage,--disable-gpu
      # ðŸ“ Storage
      STORAGE_TYPE: local
      STORAGE_LOCAL_PATH: /app/data/media
      # âš¡ Redis (deaktiviert)
      REDIS_ENABLED: "false"
      # ðŸª Webhook
      WEBHOOK_TIMEOUT: "10000"
      WEBHOOK_MAX_RETRIES: "3"
      WEBHOOK_RETRY_DELAY: "5000"
      # ðŸš¦ Rate Limiting
      RATE_LIMIT_TTL: "60"
      RATE_LIMIT_MAX: "100"
      # ðŸ”Œ Plugins
      PLUGINS_ENABLED: "true"
      PLUGINS_DIR: /app/data/plugins
      # ðŸ”‘ API Key â€” sicheren Wert setzen!
      API_MASTER_KEY: "YOUR_SECURE_API_KEY_HERE"
    volumes:
      - openwa-data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:2785/api/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - highfishNetwork

  # â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
  # ðŸ–¥ï¸ Dashboard (lokal gebaut â€” kein Build in Portainer)
  # â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•â•
  dashboard:
    image: openwa-dashboard:local    # â† Schritt 1b muss vorher ausgefÃ¼hrt werden!
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

> ðŸ”‘ **API_MASTER_KEY** â€” einen zufÃ¤lligen, sicheren Wert generieren:
> ```bash
> openssl rand -hex 32
> ```
> Diesen Key beim Dashboard-Login eingeben und als `X-API-Key`-Header in allen API-Requests verwenden.

### Nginx Proxy Manager Konfiguration

Zwei separate Proxy Hosts anlegen:

**Dashboard:**

| Feld | Wert |
|---|---|
| Domain | `YOURS` |
| Forward Hostname | `openwa-dashboard` |
| Forward Port | `80` |

**API:**

| Feld | Wert |
|---|---|
| Domain | `YOURS` |
| Forward Hostname | `openwa` |
| Forward Port | `2785` |

> ðŸ’¡ Da API und Dashboard auf **verschiedenen Subdomains** laufen, sind keine Custom Locations nÃ¶tig â€” jede Subdomain bekommt einen eigenen Proxy Host.

---

## ðŸ§ª Schritt 3 â€“ API testen

```bash
# ðŸŸ¢ Health Check (Ã¶ffentlich â€” kein Key erforderlich)
curl -i https://YOURS/api/health

# ðŸ” Authentifizierter Endpoint (Key erforderlich)
curl -i https://YOURS/api/health/detailed \
  -H "X-API-Key: YOUR_SECURE_API_KEY_HERE"
```

**âœ… Erwartete Antwort (200 OK):**
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

**âŒ Bei `401 Invalid API key`** â€” Key im Container prÃ¼fen:
```bash
docker exec -it openwa env | grep API_MASTER_KEY
# Ausgabe muss exakt dem gesetzten Key entsprechen
```

---

## ðŸ–¥ï¸ Schritt 4 â€“ Dashboard Ã¶ffnen

```
ðŸŒ  Dashboard:  https://YOURS
ðŸ”—  API-URL:    https://YOURS
ðŸ”‘  API-Key:    YOUR_SECURE_API_KEY_HERE
```

---

## ðŸ”„ Auto-Update per Cron

Das Skript prÃ¼ft tÃ¤glich ob sich etwas im `dashboard/`-Ordner geÃ¤ndert hat.
Nur dann wird neu gebaut. Alte Images werden automatisch aufgerÃ¤umt â€” es bleiben immer die **2 neuesten** erhalten.

### Skript anlegen

```bash
cat > /opt/openwa-update.sh << 'EOF'
#!/bin/bash
set -e

REPO_DIR="/opt/openwa-src"
IMAGE_NAME="openwa-dashboard"
VITE_API_URL="https://YOURS"
LOG_PREFIX="[openwa-update $(date '+%Y-%m-%d %H:%M:%S')]"

echo "$LOG_PREFIX ðŸš€ Start"

cd "$REPO_DIR"
git fetch origin main
CHANGES=$(git diff HEAD origin/main --name-only | grep "^dashboard/" || true)
git pull origin main

# ðŸ”§ Dockerfile-Fixes nach jedem git pull neu anwenden
sed -i 's/RUN npm ci/RUN npm ci --legacy-peer-deps/' dashboard/Dockerfile

if [ -z "$CHANGES" ]; then
  echo "$LOG_PREFIX âœ… Keine Dashboard-Ã„nderungen, Ã¼berspringe Build."
  exit 0
fi

echo "$LOG_PREFIX ðŸ”¨ Ã„nderungen erkannt, baue neu..."

TIMESTAMP=$(date '+%Y%m%d%H%M%S')
docker build \
  --build-arg VITE_API_URL="$VITE_API_URL" \
  -t "$IMAGE_NAME:$TIMESTAMP" \
  -t "$IMAGE_NAME:latest" \
  -t "$IMAGE_NAME:local" \
  "$REPO_DIR/dashboard"

docker restart openwa-dashboard
echo "$LOG_PREFIX â™»ï¸  Container neu gestartet."

# ðŸ§¹ Alte Images aufrÃ¤umen: nur 2 neueste behalten
echo "$LOG_PREFIX ðŸ—‘ï¸  RÃ¤ume alte Images auf..."
docker images "$IMAGE_NAME" --format "{{.Tag}}\t{{.ID}}" \
  | grep -v "latest\|local" \
  | sort -r \
  | tail -n +3 \
  | awk '{print $2}' \
  | xargs -r docker rmi

echo "$LOG_PREFIX âœ… Fertig. Aktuelle Images:"
docker images "$IMAGE_NAME"
EOF

chmod +x /opt/openwa-update.sh
```

### Cron einrichten

```bash
crontab -e
# Zeile einfÃ¼gen â€” lÃ¤uft tÃ¤glich um 3:00 Uhr:
0 3 * * * /opt/openwa-update.sh >> /var/log/openwa-update.log 2>&1
```

### Manuell testen & Logs prÃ¼fen

```bash
# Einmalig manuell ausfÃ¼hren
/opt/openwa-update.sh

# Logs live verfolgen
tail -f /var/log/openwa-update.log
```

---

## ðŸ› Bekannte Probleme & LÃ¶sungen

<details>
<summary>âŒ <b>host not found in upstream "openwa"</b></summary>

**Ursache:** Das Dashboard-`Dockerfile.traefik` nutzt einen nginx-Upstream der nur im Traefik-Netzwerk auflÃ¶sbar ist.

**LÃ¶sung:** Sicherstellen dass im Stack `image: openwa-dashboard:local` steht, **nicht** ein `build:`-Block der auf das GitHub-Repo zeigt. Das lokal gebaute Image enthÃ¤lt die richtige `nginx.conf` ohne Upstream-Proxy.
</details>

<details>
<summary>âŒ <b>npm error ERESOLVE / exit code 1</b></summary>

**Ursache:** `vite@8.x` (im Repo) ist inkompatibel mit `@vitejs/plugin-react@5.1.4` (unterstÃ¼tzt nur bis vite@7).

**LÃ¶sung:** Den `sed`-Befehl aus Schritt 1b ausfÃ¼hren â€” `npm ci --legacy-peer-deps` setzen.
</details>

<details>
<summary>âŒ <b>Error 524 / Timeout beim Portainer-Deploy</b></summary>

**Ursache:** Cloudflare und andere Proxies trennen HTTP-Verbindungen nach ~100 Sekunden. Der npm-Build dauert lÃ¤nger.

**LÃ¶sung:** **Beide** Images (API + Dashboard) **nie** Ã¼ber Portainer bauen. Immer per SSH (Schritt 1). Danach nur noch `image: openwa-api:local` und `image: openwa-dashboard:local` im Stack.
</details>

<details>
<summary>âŒ <b>401 Invalid API key</b></summary>

**Ursache:** `API_MASTER_KEY` ist leer, falsch geschrieben, oder der falsche Header wird verwendet.

**LÃ¶sung:**
```bash
# Key im laufenden Container prÃ¼fen
docker exec -it openwa env | grep API_MASTER_KEY

# Korrekter Header in curl:
curl -H "X-API-Key: DEIN_KEY" https://YOURS/api/health/detailed
```
</details>

<details>
<summary>âŒ <b>NGINX zeigt nur Standardseite</b></summary>

**Ursache:** Falsches Image oder Build nicht ausgefÃ¼hrt.

**LÃ¶sung:** Schritt 1b erneut ausfÃ¼hren, dann `docker images | grep openwa-dashboard` prÃ¼fen. Image muss vorhanden sein bevor der Stack deployed wird.
</details>

<details>
<summary>âŒ <b>npm warn deprecated ... (viele Zeilen)</b></summary>

**Ursache:** Veraltete AbhÃ¤ngigkeiten im Upstream-Repo.

**LÃ¶sung:** Keine â€” das sind nur **Warnings**, keine Fehler. Der Build lÃ¤uft trotzdem durch.
</details>

---

## ðŸ”Œ Ports

| Service | Port | URL | Beschreibung |
|---|---|---|---|
| ðŸ¤– API | `2785` | `https://YOURS/api` | REST API |
| ðŸ“– Swagger | `2785` | `https://YOURS/api/docs` | Interaktive API-Doku |
| ðŸ–¥ï¸ Dashboard | `8085` | `https://YOURS` | Web-UI (via Reverse Proxy) |
| ðŸ–¥ï¸ Dashboard | `8085` | `http://YOURS:8085` | Web-UI (LAN direkt) |

---

## ðŸ”— Links

[![OpenWA Upstream](https://img.shields.io/badge/OpenWA-Upstream%20Repo-6366f1?style=flat-square&logo=github)](https://github.com/rmyndharis/OpenWA)
[![Issues](https://img.shields.io/badge/OpenWA-Issues-ef4444?style=flat-square&logo=github)](https://github.com/rmyndharis/OpenWA/issues)
[![Portainer Docs](https://img.shields.io/badge/Portainer-Dokumentation-13BEF9?style=flat-square&logo=portainer)](https://docs.portainer.io)
[![Docker Docs](https://img.shields.io/badge/Docker-Dokumentation-2496ED?style=flat-square&logo=docker&logoColor=white)](https://docs.docker.com)

---

<div align="center">

**hAI Â· OpenWA im LAN** â€” Self-hosted, Open Source, kein Cloud-Zwang ðŸ 

<sub>Erstellt von <a href="https://github.com/jbkunama1">therealteacher</a> Â· Karlsruhe ðŸ‡©ðŸ‡ª</sub>

</div>



