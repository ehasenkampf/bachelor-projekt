# Bachelor Projekt – Infrastruktur Setup Tutorial

> **Von Hetzner Server bis zu laufendem n8n & OpenProject**  
> Schritt-für-Schritt Anleitung

| Service | URL |
|---|---|
| Landing Page | `https://bachelor-projekt.de/` |
| n8n | `https://n8n.bachelor-projekt.de/` |
| OpenProject | `https://bachelor-projekt.de/openproject/` |

---

## Inhaltsverzeichnis

1. [Überblick & Voraussetzungen](#1-überblick--voraussetzungen)
2. [Hetzner Server erstellen](#2-hetzner-server-erstellen)
3. [Domain konfigurieren](#3-domain-konfigurieren)
4. [Server einrichten](#4-server-einrichten)
5. [Projektdateien erstellen](#5-projektdateien-erstellen)
6. [nginx.conf erstellen](#6-nginxconf-erstellen)
7. [docker-compose.yml erstellen](#7-docker-composeyml-erstellen)
8. [SSL-Zertifikate (Let's Encrypt)](#8-ssl-zertifikate-lets-encrypt)
9. [Alle Services starten](#9-alle-services-starten)
10. [OpenProject SMTP in Admin-UI konfigurieren](#10-openproject-smtp-in-admin-ui-konfigurieren)
11. [Alles testen](#11-alles-testen)
12. [Troubleshooting](#troubleshooting)

---

## 1. Überblick & Voraussetzungen

### Architektur

```
Internet (HTTPS)
       |
    nginx (Reverse Proxy)
    |                |
    |                |
 n8n :5678    openproject :8080
              openproject-worker
              postgres :5432
```

### Was du brauchst

- Hetzner Cloud Account ([hetzner.com](https://hetzner.com))
- Eine Domain (z.B. bei Namecheap, Strato, etc.)
- SSH-Client (Terminal auf Mac/Linux, Windows Terminal auf Windows)
- Ca. 2–3 Stunden Zeit

### Technik-Stack

| Komponente | Details |
|---|---|
| Server | Hetzner Cloud, Ubuntu 24.04 |
| Containerisierung | Docker + Docker Compose |
| Reverse Proxy | nginx:latest |
| SSL/TLS | Let's Encrypt via Certbot |
| Workflow Automation | n8nio/n8n |
| Projektmanagement | openproject/openproject:13-slim |
| Datenbank | postgres:13 |
| SMTP | Gmail via STARTTLS Port 587 |

---

## 2. Hetzner Server erstellen

### 2.1 Account & Projekt

1. Gehe auf [hetzner.com](https://hetzner.com) und erstelle einen Account
2. Verifiziere deine E-Mail-Adresse
3. Gehe zur **Hetzner Cloud Console** → neues Projekt erstellen (z.B. "Bachelor Projekt")

### 2.2 SSH Key erstellen (lokal)

**Mac/Linux:**
```bash
ssh-keygen -t ed25519 -C "dein@email.de"
# Einfach Enter drücken bei allen Fragen

# Public Key anzeigen und kopieren:
cat ~/.ssh/id_ed25519.pub
```

**Windows (PowerShell):**
```powershell
ssh-keygen -t ed25519 -C "dein@email.de"
# Key liegt unter C:\Users\DeinName\.ssh\id_ed25519.pub
```

### 2.3 Server konfigurieren

1. In der Hetzner Console: **Server → Neuen Server erstellen**
2. Standort: `Nuremberg` (oder Falkenstein)
3. Betriebssystem: **Ubuntu 24.04**
4. Server-Typ: **CX22** (2 vCPU, 4 GB RAM) – ausreichend für dieses Projekt
5. SSH Keys: Deinen kopierten Public Key einfügen
6. Server erstellen – du bekommst eine **IP-Adresse** (z.B. `178.104.251.94`)

> ⚠️ **IP-Adresse notieren!** Sie wird in den nächsten Schritten oft gebraucht.

### 2.4 Mit Server verbinden

```bash
ssh root@178.104.251.94
# Beim ersten Mal: Fingerprint bestätigen mit 'yes'
```

✅ Du siehst einen Prompt wie: `root@ubuntu-xxxx:~#`

---

## 3. Domain konfigurieren

Füge folgende **A-Records** bei deinem Domain-Anbieter hinzu:

| DNS Record | Wert |
|---|---|
| `A   bachelor-projekt.de` | `178.104.251.94` (deine IP) |
| `A   n8n.bachelor-projekt.de` | `178.104.251.94` (gleiche IP) |

> ⏳ DNS-Änderungen brauchen 5–30 Minuten (manchmal bis zu 24h).  
> Prüfen mit: `ping bachelor-projekt.de` oder [dnschecker.org](https://dnschecker.org)

### DNS prüfen

```bash
# Auf deinem lokalen Computer:
ping bachelor-projekt.de
# Erwartete Ausgabe: Antwort von 178.104.251.94

# Oder auf dem Server:
nslookup bachelor-projekt.de
nslookup n8n.bachelor-projekt.de
```

---

## 4. Server einrichten

### 4.1 System updaten

```bash
apt update && apt upgrade -y
# Dauert ca. 2-5 Minuten
```

### 4.2 Docker installieren

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Prüfen:
docker compose version
# Erwartete Ausgabe: Docker Compose version v2.x.x
```

> ⚠️ Wir verwenden `docker compose` (mit Leerzeichen) – das moderne Plugin.  
> `docker-compose` (mit Bindestrich) ist die alte Version – nicht verwenden.

### 4.3 Docker Netzwerke erstellen

```bash
docker network create frontend-net
docker network create backend-net

# Prüfen:
docker network ls
```

### 4.4 Projektverzeichnis anlegen

```bash
mkdir -p ~/projects/infra
cd ~/projects/infra
```

---

## 5. Projektdateien erstellen

### 5.1 .env Datei (Secrets)

```bash
nano .env
```

Inhalt (Werte anpassen!):

```env
POSTGRES_PASSWORD=sicheres_passwort_hier
OPENPROJECT_SECRET_KEY_BASE=langer_zufaelliger_string_64_zeichen
SMTP_USER=deine@gmail.com
SMTP_PASS=xxxx xxxx xxxx xxxx
```

**OPENPROJECT_SECRET_KEY_BASE generieren:**
```bash
openssl rand -hex 32
```

**Gmail App-Passwort erstellen:**
1. [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
2. 2-Faktor-Authentifizierung muss aktiv sein
3. App: Andere (beliebiger Name z.B. "n8n")
4. Das generierte 16-stellige Passwort als `SMTP_PASS` eintragen

### 5.2 .gitignore erstellen

```bash
nano .gitignore
```

Inhalt:
```
.env
certbot/
```

### 5.3 Assets-Verzeichnis

```bash
mkdir -p assets
# Eigene PNG-Logos für n8n und OpenProject hier ablegen:
# assets/n8n_logo.png
# assets/openproject_logo.png
```

### 5.4 Landing Page (index.html)

```bash
nano index.html
```

<details>
<summary>index.html Inhalt anzeigen</summary>

```html
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <title>Bachelor Projekt</title>
    <style>
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #1e1e2f, #3a3a5f);
            color: white;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            text-align: center;
        }
        .container { max-width: 500px; width: 90%; }
        h1 { margin-bottom: 40px; }
        .btn {
            display: block;
            margin: 20px 0;
            border-radius: 15px;
            overflow: hidden;
            transition: 0.3s;
            background: linear-gradient(135deg, #111, #222);
        }
        .btn img { width: 100%; height: 140px; object-fit: cover; display: block; }
        .btn:hover { transform: scale(1.05); box-shadow: 0 10px 30px rgba(0,0,0,0.5); }
    </style>
</head>
<body>
<div class="container">
    <h1>Bachelor Projekt</h1>
    <a href="https://n8n.bachelor-projekt.de/" class="btn">
        <img src="/assets/n8n_logo.png" alt="n8n">
    </a>
    <a href="/openproject/" class="btn">
        <img src="/assets/openproject_logo.png" alt="OpenProject">
    </a>
</div>
</body>
</html>
```

</details>

---

## 6. nginx.conf erstellen

```bash
nano nginx.conf
```

```nginx
events {
    worker_connections 1024;
}

http {
    # ========================
    # HTTP -> HTTPS Redirect
    # ========================
    server {
        listen 80;
        server_name bachelor-projekt.de n8n.bachelor-projekt.de;

        location / {
            return 301 https://$host$request_uri;
        }

        # Certbot Challenge (SSL-Erneuerung)
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }

    # ========================
    # bachelor-projekt.de (HTTPS)
    # ========================
    server {
        listen 443 ssl;
        server_name bachelor-projekt.de;

        ssl_certificate /etc/letsencrypt/live/bachelor-projekt.de/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/bachelor-projekt.de/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Landing Page
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        # OpenProject Redirect
        location = /openproject {
            return 301 /openproject/;
        }

        # OpenProject Proxy
        location /openproject/ {
            proxy_pass http://openproject:8080/openproject/;
            proxy_http_version 1.1;
            proxy_read_timeout 300s;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Script-Name /openproject;
        }
    }

    # ========================
    # n8n.bachelor-projekt.de (HTTPS)
    # ========================
    server {
        listen 443 ssl;
        server_name n8n.bachelor-projekt.de;

        ssl_certificate /etc/letsencrypt/live/n8n.bachelor-projekt.de/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/n8n.bachelor-projekt.de/privkey.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        location / {
            proxy_pass http://n8n:5678/;
            proxy_http_version 1.1;
            proxy_read_timeout 300s;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Accept-Encoding "";
        }
    }
}
```

---

## 7. docker-compose.yml erstellen

```bash
nano docker-compose.yml
```

```yaml
services:

  # ========================
  # nginx Reverse Proxy
  # ========================
  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    networks:
      - frontend-net
      - backend-net
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./index.html:/usr/share/nginx/html/index.html
      - ./assets:/usr/share/nginx/html/assets
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
    depends_on:
      - n8n
      - openproject

  # ========================
  # n8n
  # ========================
  n8n:
    image: n8nio/n8n
    container_name: n8n
    restart: always
    networks:
      - backend-net
    volumes:
      - n8n-data:/home/node/.n8n
    environment:
      - N8N_HOST=n8n.bachelor-projekt.de
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_EDITOR_BASE_URL=https://n8n.bachelor-projekt.de/
      - WEBHOOK_URL=https://n8n.bachelor-projekt.de/
      - N8N_SECURE_COOKIE=true
      - N8N_PROXY_HOPS=1
      - N8N_EMAIL_MODE=smtp
      - N8N_SMTP_HOST=smtp.gmail.com
      - N8N_SMTP_PORT=587
      - N8N_SMTP_USER=${SMTP_USER}
      - N8N_SMTP_PASS=${SMTP_PASS}
      - N8N_SMTP_SSL=false
      - N8N_SMTP_STARTTLS=true
      - N8N_SMTP_SENDER=${SMTP_USER}

  # ========================
  # PostgreSQL Datenbank
  # ========================
  db:
    image: postgres:13
    container_name: openproject-db
    restart: always
    environment:
      POSTGRES_DB: openproject
      POSTGRES_USER: openproject
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - openproject-db-data:/var/lib/postgresql/data
    networks:
      - backend-net

  # ========================
  # OpenProject Web
  # ========================
  openproject:
    image: openproject/openproject:13-slim
    container_name: openproject
    restart: always
    depends_on:
      - db
    environment:
      OPENPROJECT_SECRET_KEY_BASE: ${OPENPROJECT_SECRET_KEY_BASE}
      OPENPROJECT_HOST__NAME: bachelor-projekt.de
      OPENPROJECT_HTTPS: "true"
      OPENPROJECT_RELATIVE_URL_ROOT: /openproject
      RAILS_RELATIVE_URL_ROOT: /openproject
      DATABASE_URL: postgres://openproject:${POSTGRES_PASSWORD}@db:5432/openproject
      OPENPROJECT_MAILER__DELIVERY__METHOD: smtp
      OPENPROJECT_SMTP__ADDRESS: smtp.gmail.com
      OPENPROJECT_SMTP__PORT: "587"
      OPENPROJECT_SMTP__ENABLE__STARTTLS__AUTO: "true"
      OPENPROJECT_SMTP__USER__NAME: ${SMTP_USER}
      OPENPROJECT_SMTP__PASSWORD: ${SMTP_PASS}
      OPENPROJECT_SMTP__AUTHENTICATION: plain
      OPENPROJECT_MAIL__FROM: ${SMTP_USER}
    networks:
      - backend-net

  # ========================
  # OpenProject Worker (für E-Mails!)
  # ========================
  openproject-worker:
    image: openproject/openproject:13-slim
    container_name: openproject-worker
    restart: always
    depends_on:
      - db
      - openproject
    command: "./docker/prod/worker"
    environment:
      OPENPROJECT_SECRET_KEY_BASE: ${OPENPROJECT_SECRET_KEY_BASE}
      OPENPROJECT_HOST__NAME: bachelor-projekt.de
      OPENPROJECT_HTTPS: "true"
      OPENPROJECT_RELATIVE_URL_ROOT: /openproject
      RAILS_RELATIVE_URL_ROOT: /openproject
      DATABASE_URL: postgres://openproject:${POSTGRES_PASSWORD}@db:5432/openproject
      OPENPROJECT_MAILER__DELIVERY__METHOD: smtp
      OPENPROJECT_SMTP__ADDRESS: smtp.gmail.com
      OPENPROJECT_SMTP__PORT: "587"
      OPENPROJECT_SMTP__ENABLE__STARTTLS__AUTO: "true"
      OPENPROJECT_SMTP__USER__NAME: ${SMTP_USER}
      OPENPROJECT_SMTP__PASSWORD: ${SMTP_PASS}
      OPENPROJECT_SMTP__AUTHENTICATION: plain
      OPENPROJECT_MAIL__FROM: ${SMTP_USER}
    networks:
      - backend-net

networks:
  frontend-net:
    external: true
  backend-net:
    external: true

volumes:
  openproject-db-data:
  n8n-data:
```

---

## 8. SSL-Zertifikate (Let's Encrypt)

### 8.1 Certbot Verzeichnisse anlegen

```bash
mkdir -p certbot/www certbot/conf
```

### 8.2 Temporäres nginx für Certbot starten

```bash
docker run --rm -d \
  --name nginx-temp \
  -p 80:80 \
  -v $(pwd)/certbot/www:/var/www/certbot \
  nginx:latest
```

### 8.3 Zertifikate anfordern

```bash
# Zertifikat für Hauptdomain
docker run --rm \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  -v $(pwd)/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  -d bachelor-projekt.de \
  --email deine@email.de \
  --agree-tos \
  --no-eff-email

# Zertifikat für n8n Subdomain
docker run --rm \
  -v $(pwd)/certbot/conf:/etc/letsencrypt \
  -v $(pwd)/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  -d n8n.bachelor-projekt.de \
  --email deine@email.de \
  --agree-tos \
  --no-eff-email
```

✅ Erfolg sieht so aus:
```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/bachelor-projekt.de/fullchain.pem
```

### 8.4 Temporäres nginx stoppen

```bash
docker stop nginx-temp
```

---

## 9. Alle Services starten

### 9.1 Dateienstruktur prüfen

```bash
ls -la ~/projects/infra/

# Sollte zeigen:
# docker-compose.yml
# nginx.conf
# index.html
# .env
# .gitignore
# assets/
# certbot/
```

### 9.2 Services starten

```bash
cd ~/projects/infra/
docker compose up -d
# Beim ersten Start: alle Images werden heruntergeladen (3-10 Minuten)
```

### 9.3 Status prüfen

```bash
docker compose ps

# Erwartete Ausgabe:
# NAME                  STATUS
# nginx                 Up
# n8n                   Up
# openproject           Up
# openproject-db        Up
# openproject-worker    Up
```

### 9.4 Logs bei Problemen

```bash
# Alle Logs
docker compose logs -f

# Nur ein Service
docker compose logs -f openproject
docker compose logs -f n8n

# Beenden mit Ctrl+C
```

---

## 10. OpenProject SMTP in Admin-UI konfigurieren

> ⚠️ **Wichtig:** OpenProject 13 benötigt die SMTP-Einstellungen **sowohl** in den Umgebungsvariablen (docker-compose.yml) **als auch** in der Admin-Oberfläche. Ohne diesen Schritt werden Einladungs-E-Mails nie verschickt.

1. Öffne: `https://bachelor-projekt.de/openproject/`
2. Einloggen als Admin (Standard: `admin` / `admin` → **sofort ändern!**)
3. Navigation: **Administration → E-Mail**
4. Folgende Werte eintragen:

| Feld | Wert |
|---|---|
| Email delivery method | `smtp` |
| SMTP server | `smtp.gmail.com` |
| SMTP port | `587` |
| SMTP authentication | `plain` |
| SMTP username | `deine@gmail.com` |
| SMTP password | App-Passwort (`xxxx xxxx xxxx xxxx`) |
| STARTTLS | ✅ aktiviert |
| SSL | ❌ deaktiviert |

5. Speichern und Test-E-Mail senden

> 📬 E-Mail landet im Spam? Das ist normal bei Gmail als Absender. Spam-Ordner prüfen und als "kein Spam" markieren.

---

## 11. Alles testen

### Browser-Tests

| URL | Erwartetes Ergebnis |
|---|---|
| `https://bachelor-projekt.de/` | Landing Page mit n8n & OpenProject Buttons |
| `https://n8n.bachelor-projekt.de/` | n8n Login/Setup Seite |
| `https://bachelor-projekt.de/openproject/` | OpenProject Login Seite |
| `http://bachelor-projekt.de/` | Redirect auf `https://...` (301) |

### SSL prüfen

```bash
curl -vI https://bachelor-projekt.de 2>&1 | grep -E 'SSL|expire|subject'
# Oder im Browser: Schloss-Symbol anklicken
```

### Container Status

```bash
docker compose ps
# Alle sollten 'Up' zeigen
# openproject-worker ist wichtig für E-Mails!
```

---

## Troubleshooting

| Problem | Lösung |
|---|---|
| `502 Bad Gateway` | Container noch nicht fertig. Warten und neu laden. `docker compose logs openproject` |
| SSL Fehler / Zertifikat nicht gefunden | Certbot hat kein Zertifikat erstellt. Schritt 8 wiederholen. `ls certbot/conf/live/` |
| n8n White Screen | Cache leeren (Strg+Shift+R). `docker compose logs n8n` |
| OpenProject lädt ewig | Erster Start dauert 3–5 Minuten. Worker muss laufen. |
| E-Mails werden nicht verschickt | Worker prüfen: `docker compose ps openproject-worker`. SMTP auch in Admin-UI eintragen (Schritt 10). |
| Port 80/443 bereits belegt | `lsof -i :80` dann `kill -9 PID` |
| DNS nicht erreichbar | Warten (bis zu 24h). Mit `nslookup` prüfen. |
| `.env` Werte werden nicht gelesen | Kein Leerzeichen um `=` herum. `cat .env` prüfen. |

### Nützliche Befehle

```bash
# Alle Container neustarten
docker compose down && docker compose up -d

# Einzelnen Container neustarten
docker compose restart openproject

# Disk-Speicher prüfen
df -h

# RAM-Auslastung prüfen
free -h

# Container Resource-Auslastung
docker stats

# In Container einloggen
docker exec -it openproject bash

# Netzwerke prüfen
docker network ls
docker network inspect backend-net
```

---

## Zusammenfassung

| Komponente | Status / Details |
|---|---|
| Hetzner Server | Ubuntu 24.04, CX22 |
| Docker | Installiert, `frontend-net` + `backend-net` |
| SSL/HTTPS | Let's Encrypt, 90 Tage gültig |
| nginx | Reverse Proxy, HTTP→HTTPS Redirect |
| n8n | `https://n8n.bachelor-projekt.de/` |
| OpenProject | `https://bachelor-projekt.de/openproject/` |
| PostgreSQL | Daten in Volume `openproject-db-data` |
| SMTP | Gmail via STARTTLS Port 587 |

> ⚠️ **Wichtige Hinweise:**
> - SSL-Zertifikate alle 90 Tage manuell erneuern
> - Standard-Admin-Passwort in OpenProject sofort ändern
> - `.env` Datei **niemals** in Git committen
> - Regelmäßige Backups der PostgreSQL Datenbank empfohlen

---

**Viel Erfolg beim Bachelor Projekt! 🚀**
