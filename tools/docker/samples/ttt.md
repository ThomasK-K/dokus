Alles klar 👍
Wir ersetzen **Traefik** durch **NGINX** als Reverse Proxy mit Let's Encrypt via Certbot.

Ich gebe dir eine saubere, production-taugliche Variante mit:

* HTTP → HTTPS Redirect
* Let's Encrypt (Webroot)
* interne Netzwerke bleiben erhalten
* keine unnötigen Ports offen


# 📁 Projektstruktur

```
text
/opt/myapp/
│
├── docker-compose.yml
├── nginx/
│   └── conf.d
│   └── certs
│   └── html
├── secrets/
│   ├── db_root_password.txt
│   └── db_password.txt
└── api/
    └── Dockerfile
```


---

# 🔥 Neue `docker-compose.yml` (mit NGINX)

```yaml
version: "3.9"

services:

  nginx:
    image: nginx:1.25-alpine
    container_name: nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/www:/var/www/certbot
      - ./nginx/certbot:/etc/letsencrypt
    networks:
      - frontend
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

  certbot:
    image: certbot/certbot:latest
    container_name: myapp-certbot
    volumes:
      - ./nginx/www:/var/www/certbot
      - ./nginx/certbot:/etc/letsencrypt
    entrypoint: >
      sh -c "trap exit TERM;
             while :; do
               certbot renew;
               sleep 12h & wait $${!};
             done"
    networks:
      - frontend

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

# 📁 NGINX Konfiguration

## `./nginx/conf.d/api.conf`

```nginx
server {
    listen 80;
    server_name api.deinedomain.de;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name api.deinedomain.de;

    ssl_certificate /etc/letsencrypt/live/api.deinedomain.de/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.deinedomain.de/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://api:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# 🔐 Erstes Zertifikat holen

Einmalig ausführen:

```bash
docker compose run --rm certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email deine@email.de \
  --agree-tos \
  --no-eff-email \
  -d api.deinedomain.de
```

Danach:

```bash
docker compose restart nginx
```

---

# 🔒 Sicherheits-Verbesserung (optional aber empfehlenswert)

In den HTTPS-Block ergänzen:

```nginx
add_header X-Frame-Options "DENY";
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "no-referrer-when-downgrade";
```

---

# 🧠 Architektur jetzt

```
Internet
   ↓
NGINX
   ↓
API
   ↓
MariaDB (internal only)
```

Genauso sauber getrennt wie vorher – nur ohne automatisches Docker-Routing.

---

# ⚖️ Traefik vs NGINX – Kurzvergleich

| Feature               | Traefik   | NGINX     |
| --------------------- | --------- | --------- |
| Auto Docker Discovery | ✅         | ❌         |
| Einfachheit           | sehr hoch | mittel    |
| Kontrolle über Config | mittel    | sehr hoch |
| Performance           | sehr gut  | exzellent |

Wenn du maximale Kontrolle willst → NGINX ist perfekt.
Wenn du viele Services dynamisch routen willst → Traefik ist komfortabler.

---

Wenn du möchtest, kann ich dir noch:

* 🔥 eine komplett gehärtete NGINX-SSL Config (A+ bei SSL Labs)
* 🔥 Rate Limiting + DDoS Basic Protection
* 🔥 HTTP/3 Setup
* 🔥 oder ein automatisches Backup + Restore Setup für MariaDB bauen

Was ist dein Ziel – kleines Projekt oder richtige Production-Umgebung?
