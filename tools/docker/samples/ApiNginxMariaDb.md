Perfekt 👍
Hier ist ein **produktionsreifes, gehärtetes `docker compose` Template** für:

* ✅ API (Node / Backend)
* ✅ MariaDB
* ✅ Reverse Proxy (Nginx)
* ✅ Datei-basierte Secrets
* ✅ internes Netzwerk
* ✅ Non-Root Container
* ✅ keine offenen DB-Ports
* ✅ read-only mounts
* ✅ restart policies

Optimiert für dein **Ubuntu 24 Server Setup**.

---

# 🔐 Projektstruktur

```
myapp/
│
├── docker-compose.yml
├── .env
├── secrets/
│   ├── db_root_password.txt
│   └── db_password.txt
├── nginx/
│   └── nginx.conf
└── api/
    └── Dockerfile
```

---

# 🔐 1️⃣ Secrets anlegen (Host)

```bash
sudo mkdir -p /opt/myapp/secrets
cd /opt/myapp/secrets

sudo nano db_root_password.txt
sudo nano db_password.txt

sudo chown root:root *.txt
sudo chmod 600 *.txt
```

---

# 🔐 2️⃣ docker-compose.yml

```yaml
version: "3.9"

services:
  mariadb:
    image: mariadb:11
    container_name: myapp-db
    restart: unless-stopped
    networks:
      - backend
    volumes:
      - db_data:/var/lib/mysql
      - /opt/myapp/secrets/db_root_password.txt:/run/secrets/db_root_password:ro
      - /opt/myapp/secrets/db_password.txt:/run/secrets/db_password:ro
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
      - /opt/myapp/secrets/db_password.txt:/run/secrets/db_password:ro
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

  nginx:
    image: nginx:1.27-alpine
    container_name: myapp-nginx
    restart: unless-stopped
    ports:
      - "80:80"
    networks:
      - frontend
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL

networks:
  backend:
    internal: true
  frontend:

volumes:
  db_data:
```

---

# 🔐 3️⃣ Nginx Config

`nginx/nginx.conf`

```nginx
events {}

http {
    server {
        listen 80;

        location / {
            proxy_pass http://api:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

---

# 🔐 4️⃣ API Dockerfile (Non-Root!)

`api/Dockerfile`

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

# 🔐 5️⃣ Passwort im Node Backend lesen

```js
import fs from "fs";

const dbPassword = fs
  .readFileSync(process.env.DB_PASSWORD_FILE, "utf8")
  .trim();
```

---

# 🛡 Sicherheitsfeatures in diesem Setup

| Feature                   | Schutz                              |
| ------------------------- | ----------------------------------- |
| internes Backend Netzwerk | DB nicht von außen erreichbar       |
| kein DB Port exposed      | Schutz vor direkten Angriffen       |
| read_only Container       | verhindert Malware Write            |
| tmpfs für temporäre Daten | kein persistentes Schreiben         |
| cap_drop ALL              | keine Linux Capabilities            |
| no-new-privileges         | verhindert Privilege Escalation     |
| Secrets als Files         | nicht via `docker inspect` sichtbar |
| Non-root API User         | keine Root-Rechte                   |

---

# 🚀 Optional (Production Upgrade)

Wenn du es **richtig professionell** willst:

* HTTPS via Let's Encrypt + Certbot
* Fail2Ban auf Host
* Firewall (ufw)
* Automatische Backups
* oder später Migration zu:

  * Traefik
  * HashiCorp Vault

---

# 📌 Wichtige Empfehlung für dich

Da du Backend + MariaDB entwickelst:

👉 Niemals DB-Port veröffentlichen
👉 Niemals Root-User für API
👉 Niemals Passwörter in `.env`

---

Wenn du willst, baue ich dir als nächstes:

* 🔒 Version mit HTTPS + Auto Let's Encrypt
* 💾 Automatisches Backup Script für MariaDB
* 🔄 Rolling Update Setup ohne Downtime
* 🧱 oder eine Variante mit Traefik statt Nginx

Was möchtest du als nächstes härten?
