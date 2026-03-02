Perfekt 👌
Dann bauen wir dir eine **saubere Staging + Production CI/CD Architektur**, professionell und skalierbar.

Stack:

* GitHub Actions
* Docker
* Docker Hub
* Nginx Reverse Proxy
* Ubuntu 24 Server
* Zwei getrennte Deploy-Umgebungen

---

# 🏗 Zielarchitektur

```text
                GitHub

        main branch  ─────►  Production
        develop      ─────►  Staging
```

---

# 🌍 Domains

| Umgebung   | Domain                  |
| ---------- | ----------------------- |
| Production | api.example.com         |
| Staging    | staging-api.example.com |

Beide laufen auf demselben Server – aber:

* getrennte Container
* getrennte Datenbanken
* getrennte ENV
* getrennte Compose Files

---

# 📂 Server Struktur

```bash
/opt/
   ├── production/
   │     └── docker-compose.yml
   └── staging/
         └── docker-compose.yml
```

---

# 1️⃣ Docker Compose – Production

`/opt/production/docker-compose.yml`

```yaml
version: "3.9"

services:
  api:
    image: deinuser/my-api:prod
    container_name: api_prod
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - frontend_net
      - backend_prod

networks:
  frontend_net:
    external: true
  backend_prod:
    external: true
```

---

# 2️⃣ Docker Compose – Staging

`/opt/staging/docker-compose.yml`

```yaml
version: "3.9"

services:
  api:
    image: deinuser/my-api:staging
    container_name: api_staging
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - frontend_net
      - backend_staging

networks:
  frontend_net:
    external: true
  backend_staging:
    external: true
```

---

# 3️⃣ Nginx Reverse Proxy

```nginx
server {
    listen 443 ssl;
    server_name api.example.com;

    location / {
        proxy_pass http://api_prod:3000;
    }
}

server {
    listen 443 ssl;
    server_name staging-api.example.com;

    location / {
        proxy_pass http://api_staging:3000;
    }
}
```

---

# 4️⃣ CI/CD Pipeline mit Branch-Logik

`.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches:
      - main
      - develop

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Set Environment
        run: |
          if [[ "${GITHUB_REF##*/}" == "main" ]]; then
            echo "IMAGE_TAG=prod" >> $GITHUB_ENV
            echo "TARGET_DIR=/opt/production" >> $GITHUB_ENV
          else
            echo "IMAGE_TAG=staging" >> $GITHUB_ENV
            echo "TARGET_DIR=/opt/staging" >> $GITHUB_ENV
          fi

      - name: Build Image
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/my-api:${{ env.IMAGE_TAG }} .

      - name: Push Image
        run: docker push ${{ secrets.DOCKER_USERNAME }}/my-api:${{ env.IMAGE_TAG }}

      - name: Deploy
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd ${{ env.TARGET_DIR }}
            docker compose pull
            docker compose up -d
            docker image prune -f
```

---

# 🔐 Environment Variablen trennen

Production `.env`:

```env
NODE_ENV=production
DB_HOST=mariadb_prod
```

Staging `.env`:

```env
NODE_ENV=staging
DB_HOST=mariadb_staging
```

---

# 🔁 Workflow

| Aktion             | Ergebnis                     |
| ------------------ | ---------------------------- |
| Push auf `develop` | Staging wird aktualisiert    |
| Push auf `main`    | Production wird aktualisiert |

---

# 🛡 Optional – Professioneller Aufbau

## 🔵 GitHub Environments

Du kannst:

* „staging“ Environment
* „production“ Environment mit Approval Required

So wird Production erst live nach manuellem Klick.

---

# 🚀 Enterprise-Upgrade möglich

Wenn du willst, kann ich dir zusätzlich bauen:

* Blue/Green nur für Production
* Datenbank-Migrations-Strategie
* Healthchecks + automatisches Rollback
* Canary Deployment
* Separate Server für Staging
* Auto-Tagging mit SemVer

---

# 🎯 Empfehlung für dich

Da du Backend + DB + Reverse Proxy nutzt:

👉 Staging auf gleichem Server ist okay für kleine/mittlere Projekte
👉 Für Business-kritisch: 2 getrennte Server

---

Willst du als Nächstes:

* 🔵 Blue/Green Production?
* 🛡 Rollback-System?
* 📦 Multi-Service Setup (API + Worker + Redis)?
* ☁️ Cloud-ready Architektur?

Sag mir, wie „enterprise“ das werden soll 😄
