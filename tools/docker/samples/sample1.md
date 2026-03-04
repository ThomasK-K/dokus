Perfekt 👍 Dann bauen wir ein **komplettes Rootless Multi-API Setup mit Nginx + HTTPS** auf Ubuntu 24.

Es enthält:

* 3 APIs (Users, Billing, Files)
* MariaDB pro API
* Nginx Reverse Proxy
* Rootless / Non-Root Container
* Datei-basierte Secrets
* Read-only Mounts, cap_drop ALL, no-new-privileges

---

# 📁 Projektstruktur

```text
/opt/myapp/
│
├── nginx/
│   ├── docker-compose.yml
│   ├── conf.d/
│   │   ├── users.conf
│   │   ├── billing.conf
│   │   └── files.conf
│   └── certs/   # hier liegen die Let's Encrypt Zertifikate
├── users-api/
│   ├── docker-compose.yml
│   └── Dockerfile
├── billing-api/
│   ├── docker-compose.yml
│   └── Dockerfile
├── files-api/
│   ├── docker-compose.yml
│   └── Dockerfile
└── shared-secrets/
    ├── db_root_password.txt
    └── db_password.txt
```

---

# 🔐 1️⃣ Rootless Docker vorbereiten

1. Rootless Docker installieren:

```bash
curl -fsSL https://get.docker.com/rootless | sh
export PATH=$HOME/bin:$PATH
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
```

2. Systemd User Service aktivieren:

```bash
systemctl --user enable docker
systemctl --user start docker
sudo loginctl enable-linger $USER
```

Prüfen:

```bash
docker info | grep Rootless
# sollte rootless: true anzeigen
```

---

# 🔐 2️⃣ Nginx Compose (Reverse Proxy)

`/opt/myapp/nginx/docker-compose.yml`

```yaml
version: "3.9"

services:
  nginx:
    image: nginx:1.27-alpine
    container_name: nginx-proxy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
    networks:
      - users-network
      - billing-network
      - files-network
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

networks:
  users-network:
    external: true
  billing-network:
    external: true
  files-network:
    external: true
```

---

# 🔐 3️⃣ Nginx Config Beispiel für Users API

`nginx/conf.d/users.conf`

```nginx
server {
    listen 80;
    server_name api.users.deinedomain.de;
    location / { return 301 https://$host$request_uri; }
}

server {
    listen 443 ssl;
    server_name api.users.deinedomain.de;

    ssl_certificate /etc/nginx/certs/users.crt;
    ssl_certificate_key /etc/nginx/certs/users.key;

    location / {
        proxy_pass http://users-api:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> Analog für Billing + Files API

---

# 🔐 4️⃣ Users API Compose

`/opt/myapp/users-api/docker-compose.yml`

```yaml
version: "3.9"

services:
  users-db:
    image: mariadb:11
    restart: unless-stopped
    networks:
      - users-network
    volumes:
      - db_data:/var/lib/mysql
      - ../../shared-secrets/db_root_password.txt:/run/secrets/root:ro
      - ../../shared-secrets/db_password.txt:/run/secrets/db:ro
    environment:
      MARIADB_ROOT_PASSWORD_FILE: /run/secrets/root
      MARIADB_DATABASE: users
      MARIADB_USER: users
      MARIADB_PASSWORD_FILE: /run/secrets/db
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true

  users-api:
    build: .
    restart: unless-stopped
    networks:
      - users-network
    depends_on:
      - users-db
    read_only: true
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    user: "1000:1000"

networks:
  users-network:
    external: true

volumes:
  db_data:
```

> Analog für Billing + Files API

---

# 🔐 5️⃣ API Dockerfile Beispiel

`users-api/Dockerfile`

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .

RUN addgroup -g 1000 appgroup \
 && adduser -D -u 1000 -G appgroup appuser

USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
```

---

# 🔐 6️⃣ HTTPS Zertifikate (Let's Encrypt)

1. Host-Ports 80/443 offen
2. Certbot Container starten:

```bash
docker run -it --rm \
  -v /opt/myapp/nginx/certs:/etc/letsencrypt \
  certbot/certbot certonly \
  --standalone \
  -d api.users.deinedomain.de \
  -d api.billing.deinedomain.de \
  -d api.files.deinedomain.de
```

3. Zertifikate landen in `nginx/certs/` → Nginx mountet read-only

---

# 🔐 7️⃣ Netzwerke erstellen

```bash
docker network create users-network
docker network create billing-network
docker network create files-network
```

---

# 🔥 Vorteile

* Rootless Docker für alle APIs
* cap_drop ALL + Non-Root Container
* Read-only Mounts + tmpfs
* Multi-App Isolation
* Nginx zentral für HTTPS + Reverse Proxy
* Skalierbar → `docker compose up -d --scale users-api=3`

---

Wenn du willst, kann ich als nächstes eine **erweiterte Version mit automatischem DB-Backup + HTTPS HSTS + HTTP → HTTPS Redirect** bauen, sodass die Multi-App Umgebung **vollständig production-ready** ist.

Willst du, dass ich das mache?
