Perfekt 👍 Dann bauen wir eine **produktive Docker-Struktur mit**

* Reverse Proxy: Nginx
* ACME DNS-01 über IPv64
* Zertifikate via Certbot
* Container-Verwaltung mit Portainer
* Abgesichertes Dashboard
* Saubere Netzwerk-Trennung (Produktionsstruktur)

Das ist eine robuste Server-Architektur – geeignet für VPS oder dedizierten Server.

---

# 🏗 Zielarchitektur (Produktion)

```
                Internet
                    │
               Firewall (UFW)
                    │
                Nginx (Host)
         TLS + Reverse Proxy + HSTS
                    │
         ┌──────────┴──────────┐
         │                     │
     frontend_net          backend_net
         │                     │
    App / API           MariaDB / Redis
         
         management_net (intern)
                │
            Portainer
```

**Wichtige Prinzipien:**

* 🔒 Nur Port 443 offen
* 🔐 MariaDB nicht öffentlich erreichbar
* 🔐 Portainer nur intern + BasicAuth
* 🔄 Auto-Renew mit Deploy-Hook
* 🧱 Getrennte Docker-Netzwerke

---

# 1️⃣ Docker Netzwerke anlegen

```bash
docker network create frontend_net
docker network create backend_net
docker network create management_net
```

---

# 2️⃣ Docker Compose – Produktionsstruktur

`docker-compose.yml`

```yaml
version: "3.9"

services:

  # =========================
  # Portainer (Management)
  # =========================
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    command: -H unix:///var/run/docker.sock
    networks:
      - management_net
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data

  # =========================
  # Beispiel API
  # =========================
  api:
    image: node:20-alpine
    container_name: api
    restart: unless-stopped
    networks:
      - frontend_net
      - backend_net
    environment:
      - DB_HOST=mariadb
    depends_on:
      - mariadb

  # =========================
  # MariaDB (nur intern)
  # =========================
  mariadb:
    image: mariadb:11
    container_name: mariadb
    restart: unless-stopped
    networks:
      - backend_net
    environment:
      - MYSQL_ROOT_PASSWORD=supersecret
      - MYSQL_DATABASE=app
    volumes:
      - mariadb_data:/var/lib/mysql

networks:
  frontend_net:
    external: true
  backend_net:
    external: true
  management_net:
    external: true

volumes:
  portainer_data:
  mariadb_data:
```

⚠️ Kein Port-Mapping → alles bleibt intern.

---

# 3️⃣ Certbot DNS-Challenge (IPv64)

Zertifikat erstellen:

```bash
certbot certonly \
  --authenticator dns-multi \
  --dns-multi-credentials /root/ipv64.ini \
  --dns-multi-providers ipv64 \
  -d example.ipv64.net \
  -d '*.example.ipv64.net'
```

Zertifikate liegen dann in:

```
/etc/letsencrypt/live/example.ipv64.net/
```

---

# 4️⃣ Nginx als Reverse Proxy (Host)

`/etc/nginx/sites-available/reverse-proxy`

```nginx
server {
    listen 443 ssl http2;
    server_name example.ipv64.net api.example.ipv64.net;

    ssl_certificate /etc/letsencrypt/live/example.ipv64.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.ipv64.net/privkey.pem;

    add_header Strict-Transport-Security "max-age=63072000" always;

    location / {
        proxy_pass http://api:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

⚠️ Wichtig:
Nginx muss im Docker-Netz laufen ODER du nutzt `proxy_pass http://127.0.0.1:PORT`.

Sauberer ist:

👉 **Nginx ebenfalls als Container betreiben.**

---

# 5️⃣ Nginx als Container (empfohlen)

```yaml
  nginx:
    image: nginx:stable-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "443:443"
    networks:
      - frontend_net
      - management_net
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
```

Dann kann `proxy_pass http://api:3000;` direkt funktionieren.

---

# 6️⃣ Portainer Dashboard absichern

### Variante A (empfohlen): Nginx BasicAuth

Passwort erzeugen:

```bash
sudo apt install apache2-utils
htpasswd -c /etc/nginx/.htpasswd admin
```

Nginx Location ergänzen:

```nginx
location /portainer/ {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://portainer:9000/;
}
```

Zusätzlich:

* IP Whitelisting optional
* Kein direkter Port 9000 offen
* Nur über HTTPS erreichbar

---

# 7️⃣ Certbot Auto-Renew mit Reload

Hook:

```bash
nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Inhalt:

```bash
#!/bin/bash
docker exec nginx nginx -s reload
```

---

# 8️⃣ Firewall (sehr wichtig)

```bash
sudo ufw allow 443/tcp
sudo ufw deny 80
sudo ufw enable
```

Optional:

```bash
sudo ufw allow from DEINE_IP to any port 22
```

---

# 🔐 Sicherheits-Hardening

Für echte Produktion zusätzlich:

* Fail2ban
* CrowdSec
* Docker rootless mode
* Separate User für Container
* Read-only Filesystem
* Resource Limits
* Automatic Updates (watchtower)

---

# 🎯 Ergebnis

✔ Reverse Proxy über Nginx
✔ Wildcard via IPv64 DNS
✔ Keine offenen Container-Ports
✔ Portainer abgesichert
✔ Saubere Netzwerk-Trennung
✔ Produktionsfähig

---

Wenn du willst, kann ich dir als Nächstes:

* 🔥 eine Enterprise-Hardened Version bauen
* 🛡 mTLS zwischen Services
* 🔄 CI/CD Pipeline mit Zero-Downtime Deploy
* 📦 Multi-Server Setup mit externem Loadbalancer
* ☁️ Setup für mehrere Domains & mehrere DNS Provider

Wie produktiv soll das werden – kleines VPS oder Business-kritisch?
