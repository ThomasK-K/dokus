Perfekt 👍
Ich baue dir jetzt eine **produktionsreife CI/CD-Pipeline** für dein Docker-Setup mit:

* Git Repository
* GitHub Actions als CI
* Docker Image Build
* Docker Hub als Registry
* Deployment auf deinen Server (Ubuntu 24)
* Zero-Downtime via Rolling Container Replace
* Funktioniert mit deinem Nginx Reverse Proxy

---

# 🏗 Zielarchitektur

```text
Git Push →
GitHub Actions →
Docker Image Build →
Push zu Docker Hub →
SSH auf Server →
docker compose pull →
docker compose up -d →
Container wird ersetzt
```

---

# 1️⃣ Voraussetzungen

Auf deinem Server:

```bash
docker --version
docker compose version
```

Docker Login (einmalig):

```bash
docker login
```

---

# 2️⃣ Projektstruktur

```text
my-api/
 ├── Dockerfile
 ├── docker-compose.prod.yml
 └── .github/workflows/deploy.yml
```

---

# 3️⃣ Dockerfile (API Beispiel)

```dockerfile
FROM node:20-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000
CMD ["node", "server.js"]
```

---

# 4️⃣ docker-compose.prod.yml (Server)

```yaml
version: "3.9"

services:
  api:
    image: deinDockerHubUser/my-api:latest
    container_name: api
    restart: unless-stopped
    networks:
      - frontend_net
      - backend_net
    environment:
      - NODE_ENV=production
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 512M

networks:
  frontend_net:
    external: true
  backend_net:
    external: true
```

⚠️ Kein Build im Prod-Compose → nur Pull!

---

# 5️⃣ GitHub Secrets setzen

In GitHub → Repository → Settings → Secrets → Actions:

| Name            | Inhalt                   |
| --------------- | ------------------------ |
| DOCKER_USERNAME | dein Docker Hub Username |
| DOCKER_PASSWORD | dein Docker Hub Token    |
| SERVER_HOST     | deine Server IP          |
| SERVER_USER     | z.B. ubuntu              |
| SERVER_SSH_KEY  | dein Private SSH Key     |

---

# 6️⃣ GitHub Actions Pipeline

`.github/workflows/deploy.yml`

```yaml
name: Build & Deploy

on:
  push:
    branches: [ "main" ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build Image
        run: docker build -t ${{ secrets.DOCKER_USERNAME }}/my-api:latest .

      - name: Push Image
        run: docker push ${{ secrets.DOCKER_USERNAME }}/my-api:latest

      - name: Deploy to Server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/my-api
            docker compose -f docker-compose.prod.yml pull
            docker compose -f docker-compose.prod.yml up -d
            docker image prune -f
```

---

# 🔄 Was passiert jetzt automatisch?

Bei jedem Push auf `main`:

1. Code wird ausgecheckt
2. Docker Image gebaut
3. Image zu Docker Hub gepusht
4. Server verbindet sich per SSH
5. Neuer Container wird gezogen
6. Alter Container wird ersetzt
7. Nginx proxyt weiter → keine Downtime

---

# 🚀 Optional: Versionierte Images

Statt `latest` besser:

```yaml
docker build -t user/my-api:${{ github.sha }} .
```

Dann kannst du:

* Rollback machen
* Releases versionieren
* Staging / Production trennen

---

# 🔐 Sicherheitsverbesserungen

Für echte Produktion:

* Docker Hub Access Token statt Passwort
* SSH Key nur mit Deploy-Rechten
* UFW nur für GitHub IP Ranges öffnen
* Fail2ban aktiv
* Separate Deploy-User ohne sudo

---

# 🎯 Bonus: Zero-Downtime (Blue/Green)

Wenn du willst, baue ich dir:

* Zwei Container Slots (`api_blue`, `api_green`)
* Healthcheck
* Automatischen Switch
* Kein einziges verlorenes Request

---

# 💡 Für dein Setup besonders sinnvoll:

Da du:

* Backend APIs baust
* MariaDB nutzt
* Nginx Reverse Proxy hast
* Containerisierte Umgebung willst

Ist diese Pipeline genau der richtige nächste Schritt.

---

Willst du als Nächstes:

* 🔵 Blue/Green Deployment?
* 🟢 Staging + Production Umgebung?
* 🔐 Full Enterprise Hardened Pipeline?
* 🏢 Self-hosted CI statt GitHub?

Wie professionell soll das Setup werden?
