Perfekt 👍
Hier kommt die **Production-Variante mit HTTPS + automatischem Let's Encrypt**, sauber gehärtet und ohne manuelles Zertifikats-Handling.

Wir verwenden:

* Reverse Proxy: Traefik
* Zertifikate von: Let's Encrypt
* Docker Compose (ohne Swarm)

Warum Traefik statt Nginx?
→ automatische Zertifikate
→ automatische Erneuerung
→ kein Certbot
→ weniger Konfigurationsaufwand
→ perfekt für Compose

---

# 🔐 Voraussetzungen

* Domain (z.B. `api.deinedomain.de`)
* DNS A-Record zeigt auf deinen Server
* Ports 80 und 443 offen
* Ubuntu 24 Server

---

# 📁 Projektstruktur

```text
/opt/myapp/
│
├── docker-compose.yml
├── traefik/
│   └── acme.json
├── secrets/
│   ├── db_root_password.txt
│   └── db_password.txt
└── api/
    └── Dockerfile
```

---

# 🔐 1️⃣ acme.json vorbereiten

```bash
mkdir -p /opt/myapp/traefik
touch /opt/myapp/traefik/acme.json
chmod 600 /opt/myapp/traefik/acme.json
```

Wichtig: 600 sonst startet Traefik nicht.

---

# 🔐 2️⃣ HTTPS Compose Template

```yaml
version: "3.9"

services:

  traefik:
    image: traefik:v3.0
    container_name: myapp-traefik
    restart: unless-stopped
    command:
      - "--api.dashboard=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.le.acme.httpchallenge=true"
      - "--certificatesresolvers.le.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.le.acme.email=deine@email.de"
      - "--certificatesresolvers.le.acme.storage=/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik/acme.json:/acme.json
    networks:
      - frontend
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  mariadb:
    image: mariadb:11
    container_name: myapp-db
    restart: unless-stopped
    networks:
      - backend
    volumes:
      - db_data:/var/lib/mysql
      - ./secrets/db_root_password.txt:/run/secrets/db_root_password:ro
      - ./secrets/db_password.txt:/run/secrets/db_password:ro
    environment:
      MARIADB_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MARIADB_DATABASE: myapp
      MARIADB_USER: myappuser
      MARIADB_PASSWORD_FILE: /run/secrets/db_password
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  api:
    build: ./api
    container_name: myapp-api
    restart: unless-stopped
    depends_on:
      - mariadb
    networks:
      - backend
      - frontend
    volumes:
      - ./secrets/db_password.txt:/run/secrets/db_password:ro
    environment:
      DB_HOST: mariadb
      DB_USER: myappuser
      DB_NAME: myapp
      DB_PASSWORD_FILE: /run/secrets/db_password
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`api.deinedomain.de`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=le"
      - "traefik.http.services.myapp.loadbalancer.server.port=3000"
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    user: "1000:1000"

networks:
  backend:
    internal: true
  frontend:

volumes:
  db_data:
```

---

# 🚀 Start

```bash
cd /opt/myapp
docker compose up -d --build
```

Beim ersten Request auf:

```
https://api.deinedomain.de
```

holt Traefik automatisch das Zertifikat.

---

# 🔒 Sicherheits-Highlights

* Automatische HTTPS Zertifikate
* Auto-Renew (alle 60 Tage)
* Kein DB-Port exposed
* internes Netzwerk
* Non-Root API
* no-new-privileges
* cap_drop ALL
* Secrets als Files
* Docker Socket read-only

---

# 🛡 Extra Härtung (optional)

Wenn du maximale Sicherheit willst:

1. Firewall:

   ```bash
   sudo ufw allow 80
   sudo ufw allow 443
   sudo ufw deny 3306
   ```

2. Docker Socket Proxy statt direktem Mount

3. Rate Limiting in Traefik

4. CrowdSec oder Fail2Ban auf Host

---

# 📌 Für dein Backend-Projekt

Dieses Setup ist absolut production-ready für:

* Node API
* MariaDB
* Dokument-Upload
* Image-Upload

---

Wenn du willst, mache ich dir als nächstes:

* 🔥 Variante mit automatischem DB-Backup
* 🔒 Version mit HTTP → HTTPS Redirect + HSTS
* 🧱 Zero-Downtime Rolling Updates
* 📊 Monitoring mit Prometheus + Grafana
* 🛡 Noch stärker gehärtete Variante (Socket Proxy, Rootless Docker)

Was soll ich dir als nächstes optimieren?
