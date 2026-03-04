Alles klar 👍 Wenn du **Nginx statt Traefik** einsetzen willst, brauchen wir ein **Multi-App Setup mit Nginx als Reverse Proxy**, das für mehrere APIs gleichzeitig HTTPS bereitstellt. Ich baue dir ein **sauberes Rootless-/Secure Setup mit Docker Compose**.

---

# 🧱 Architektur

* Zentrale Nginx Instanz als Reverse Proxy + HTTPS
* Jede API in eigenem Compose-Projekt
* Getrennte interne Netzwerke für jede API
* MariaDB optional pro API oder zentral
* Datei-basierte Secrets
* Rootless- / Non-Root Container

---

# 📁 Projektstruktur (Beispiel mit 3 APIs)

```text
/opt/myapp/
│
├── nginx/
│   ├── docker-compose.yml
│   └── conf.d/
│       ├── users.conf
│       ├── billing.conf
│       └── files.conf
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

# 🔐 1️⃣ Nginx Docker Compose

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

# 🔐 2️⃣ Beispiel Nginx Config für Users API

`nginx/conf.d/users.conf`

```nginx
server {
    listen 80;
    server_name api.users.deinedomain.de;

    location / {
        return 301 https://$host$request_uri;
    }
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

> ⚠️ Für jede API erstellst du ein eigenes `.conf` + Zertifikat.

---

# 🔐 3️⃣ Users API Docker Compose

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

> Analog für Billing + Files API.

---

# 🔐 4️⃣ HTTPS Zertifikate

Da wir **Let's Encrypt nicht automatisch via Nginx** haben (wie Traefik), hast du 2 Optionen:

1. **Certbot lokal / on Host**

   * Zertifikate liegen dann unter `/opt/myapp/nginx/certs/`
   * Nginx mountet sie read-only

2. **Certbot Container + Volume** (modern)

Beispiel:

```bash
docker run -it --rm \
  -v /opt/myapp/nginx/certs:/etc/letsencrypt \
  certbot/certbot certonly \
  --standalone -d api.users.deinedomain.de
```

---

# 🔥 Vorteile dieses Setups

* Separate Netzwerke → APIs isoliert
* Non-Root Container + cap_drop ALL
* Nginx als zentraler Reverse Proxy
* Read-only Mounts für Secrets + Config
* Multi-App skalierbar: einfach weitere API + Conf + Zertifikat

---

# 🧠 Tipp für Multi-App + Nginx

* Für 10+ APIs empfehle ich **zentrale Zertifikatsstruktur + automatisches Deployment**
* Optional kannst du einen **Proxy Manager wie Nginx Proxy Manager** einsetzen, wenn du GUI willst
* Alle Apps können rootless laufen, Nginx braucht ggf. Host-Ports (80/443)

---

Ich kann dir als nächstes ein **fertiges Rootless Multi-App Docker Compose Template mit Nginx + 3 APIs + HTTPS** bauen, das sofort auf Ubuntu 24 läuft.

Willst du, dass ich das mache?
