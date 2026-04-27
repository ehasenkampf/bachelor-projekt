# Projektdokumentation – Bachelor Projekt

**Projektname:** Bachelor Projekt – Infrastruktur Setup  
**Methodik:** PRINCE2  
**Server:** Hetzner Cloud  
**Domain:** bachelor-projekt.de

---

## Arbeitspaket 0 – Server-Einrichtung bei Hetzner

### Konzept

- Bereitstellung eines Linux-Servers (Ubuntu 24) bei Hetzner Cloud
- Öffentliche IP: `178.104.251.94`
- Domain `bachelor-projekt.de` wird auf den Server gezeigt
- Docker und Docker Compose werden als Container-Laufzeitumgebung verwendet
- Netzwerke `frontend-net` und `backend-net` werden als externe Docker-Netzwerke angelegt

### Customizing

- Installation von Docker und Docker Compose auf dem Server
- Erstellung der externen Docker-Netzwerke:
  ```bash
  docker network create frontend-net
  docker network create backend-net
  ```
- DNS-Eintrag für `bachelor-projekt.de` → `178.104.251.94`
- DNS-Eintrag für `n8n.bachelor-projekt.de` → `178.104.251.94`

### Betrieb / Support

- Server läuft dauerhaft mit `restart: always` auf allen Containern
- Zugriff über SSH
- Projektverzeichnis: `~/projects/infra/`

---

## Arbeitspaket 1 – Technisches Setup nginx (Reverse Proxy)

### Konzept

- nginx wird als zentraler Reverse Proxy für alle Dienste eingesetzt
- Landing Page unter `https://bachelor-projekt.de`
- SSL/TLS-Verschlüsselung über Let's Encrypt (Certbot)
- HTTP wird automatisch auf HTTPS umgeleitet
- Separate Subdomain `n8n.bachelor-projekt.de` für n8n
- OpenProject läuft unter `https://bachelor-projekt.de/openproject/`

### Customizing

- `nginx.conf` mit folgenden Server-Blöcken:
  - Port 80: HTTP → HTTPS Redirect + Certbot Challenge
  - Port 443 `bachelor-projekt.de`: Landing Page + OpenProject
  - Port 443 `n8n.bachelor-projekt.de`: n8n Reverse Proxy
- SSL-Zertifikate erstellt mit Certbot Webroot-Methode:
  - `bachelor-projekt.de` – gültig bis 26.07.2026
  - `n8n.bachelor-projekt.de` – gültig bis 26.07.2026
- Landing Page (`index.html`) mit zwei Buttons (n8n, OpenProject) und Logo-Bildern
- `Accept-Encoding ""` Header gesetzt um doppelte gzip-Komprimierung zu verhindern
- Statische Assets (PNG-Logos) werden über exakte `location =` Blöcke direkt aus nginx geliefert

### Betrieb / Support

- nginx Container läuft mit `restart: always`
- Konfiguration liegt unter `~/projects/infra/nginx.conf`
- Zertifikate liegen unter `~/projects/infra/certbot/conf/`
- Neustart nach Konfigurationsänderungen:
  ```bash
  docker compose restart nginx
  ```

---

## Arbeitspaket 2 – Technisches Setup n8n

### Konzept

- n8n wird als Workflow-Automatisierungs-Tool für das Team bereitgestellt
- Erreichbar unter `https://n8n.bachelor-projekt.de`
- Persistente Datenspeicherung über Docker Volume
- Team-Mitglieder können per E-Mail eingeladen werden
- SMTP über Gmail für E-Mail-Versand

### Customizing

- Docker Image: `n8nio/n8n` (Version 2.17.7)
- Konfiguration über Umgebungsvariablen:
  ```
  N8N_HOST=n8n.bachelor-projekt.de
  N8N_PROTOCOL=https
  N8N_EDITOR_BASE_URL=https://n8n.bachelor-projekt.de/
  WEBHOOK_URL=https://n8n.bachelor-projekt.de/
  N8N_SECURE_COOKIE=true
  N8N_PROXY_HOPS=1
  ```
- SMTP-Konfiguration über Gmail App Password:
  ```
  N8N_EMAIL_MODE=smtp
  N8N_SMTP_HOST=smtp.gmail.com
  N8N_SMTP_PORT=587
  N8N_SMTP_STARTTLS=true
  ```
- Sensible Zugangsdaten (`SMTP_USER`, `SMTP_PASS`) in `.env` Datei ausgelagert
- Docker Volume `n8n-data` für persistente Datenspeicherung unter `/home/node/.n8n`
- Owner Account wurde nach Erststart eingerichtet

### Betrieb / Support

- n8n Container läuft mit `restart: always`
- Datenpersistenz über Docker Volume `n8n-data`
- Team-Mitglieder einladen: **Settings → Users → Invite people**
- Logs einsehen:
  ```bash
  docker logs n8n --tail 50
  ```

---

## Arbeitspaket 3 – Technisches Setup OpenProject

### Konzept

- OpenProject wird als Projektmanagement-Tool nach PRINCE2 eingesetzt
- Erreichbar unter `https://bachelor-projekt.de/openproject/`
- Eigene PostgreSQL-Datenbank für Datenpersistenz
- Subpfad-Konfiguration über `RAILS_RELATIVE_URL_ROOT`

### Customizing

- Docker Image: `openproject/openproject:13-slim`
- PostgreSQL Datenbank: `postgres:13`
- Konfiguration über Umgebungsvariablen:
  ```
  OPENPROJECT_HOST__NAME=bachelor-projekt.de
  OPENPROJECT_HTTPS=true
  OPENPROJECT_RELATIVE_URL_ROOT=/openproject
  RAILS_RELATIVE_URL_ROOT=/openproject
  ```
- Sensible Zugangsdaten (`POSTGRES_PASSWORD`, `OPENPROJECT_SECRET_KEY_BASE`) in `.env` Datei ausgelagert
- Docker Volume `openproject-db-data` für persistente Datenbankdaten
- nginx Header `X-Forwarded-Script-Name: /openproject` für korrekte Subpfad-Erkennung

### Betrieb / Support

- OpenProject Container läuft mit `restart: always`
- Datenbankpersistenz über Docker Volume `openproject-db-data`
- Logs einsehen:
  ```bash
  docker logs openproject --tail 50
  ```

---

## Sicherheit & Best Practices

- Alle sensiblen Daten (Passwörter, Secrets, SMTP-Zugangsdaten) in `.env` Datei
- `.env` und `certbot/` Verzeichnis in `.gitignore` eingetragen
- Quellcode ohne Secrets auf GitHub: `https://github.com/ehasenkampf/bachelor-projekt`
- HTTPS für alle Dienste aktiviert
- HTTP wird automatisch auf HTTPS umgeleitet
- n8n `N8N_SECURE_COOKIE=true` aktiviert

---

## Projektstruktur

```
~/projects/infra/
├── docker-compose.yml
├── nginx.conf
├── index.html
├── .env                  # nicht in Git
├── .gitignore
├── assets/
│   ├── n8n_logo.png
│   └── openproject_logo.png
└── certbot/              # nicht in Git
    ├── www/
    └── conf/
```

---

## Dienste Übersicht

| Dienst       | URL                                      | Status |
| ------------ | ---------------------------------------- | ------ |
| Landing Page | https://bachelor-projekt.de              | ✅     |
| n8n          | https://n8n.bachelor-projekt.de          | ✅     |
| OpenProject  | https://bachelor-projekt.de/openproject/ | ✅     |

---

_Dokumentation erstellt: April 2026_
