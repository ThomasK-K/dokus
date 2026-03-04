Da du:

* `read_only: true`
* `user: "1000:1000"`
* Secrets über `DB_PASSWORD_FILE`
* MariaDB im internen Netz

verwendest, zeige ich dir ein **sauberes Production-Dockerfile für eine Node.js API**.

Ich gehe davon aus, dass du Express/Fastify o. ä. nutzt und auf Port 3000 hörst.

---

# 🔥 Production Dockerfile (Node 20 Alpine, Multi-Stage)

```dockerfile
# -------- Stage 1: Build --------
FROM node:20-alpine AS builder

WORKDIR /app

# Nur package files zuerst für besseres Caching
COPY package*.json ./

# Install nur production deps später, aber hier brauchen wir alles zum builden
RUN npm ci

COPY . .

# Falls du TypeScript nutzt:
# RUN npm run build

# -------- Stage 2: Runtime --------
FROM node:20-alpine

# Security: eigene Gruppe/User mit fester UID 1000
RUN addgroup -g 1000 nodegroup \
 && adduser -D -u 1000 -G nodegroup nodeuser

WORKDIR /app

# Nur production dependencies kopieren
COPY --from=builder /app/package*.json ./
RUN npm ci --omit=dev

# Gebaute App kopieren
COPY --from=builder /app ./

# Rechte korrekt setzen
RUN chown -R nodeuser:nodegroup /app

USER 1000:1000

ENV NODE_ENV=production
EXPOSE 3000

CMD ["node", "server.js"]
```

---

# 🧠 Falls du TypeScript verwendest

Dann im Builder:

```dockerfile
RUN npm run build
```

Und im Runtime-Image:

```dockerfile
COPY --from=builder /app/dist ./dist
CMD ["node", "dist/server.js"]
```

---

# 🔐 Optional: Noch härter

## 1️⃣ Keine Shell im Runtime-Image

Statt `node:alpine` kannst du distroless verwenden:

```dockerfile
FROM gcr.io/distroless/nodejs20
```

Aber:

* Debugging wird schwerer
* kein `sh` im Container

---

## 2️⃣ HEALTHCHECK (empfohlen)

Wenn deine API `/health` hat:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1
```

(Alpine braucht `apk add --no-cache wget`)

Oder besser:

```dockerfile
HEALTHCHECK CMD node healthcheck.js
```

---

# 🧩 Wichtig wegen `read_only: true`

Da dein Container read-only läuft:

* Keine Logs in Dateien schreiben ❌
* Nur stdout verwenden ✅
* Falls Uploads → eigenes Volume mounten
* `/tmp` wird durch `tmpfs` bereitgestellt (gut!)

---

# 📁 Minimaler Projektaufbau

```
api/
 ├── Dockerfile
 ├── package.json
 ├── server.js
 └── src/
```

---

# 🚀 Performance-Optimierung

In `package.json`:

```json
{
  "scripts": {
    "start": "node server.js"
  }
}
```

Dann im Dockerfile:

```dockerfile
CMD ["npm", "start"]
```

---

# 🏆 Ergebnis

* Nicht root
* Multi-stage build
* Kleine Image Size (~90–120 MB)
* Kompatibel mit deinem Compose Setup
* Production ready

---

Wenn du willst, kann ich dir noch:

* 🔥 ein ultra-kleines 40MB Image bauen
* 🔥 eine perfekte Express Boilerplate passend zu deiner MariaDB
* 🔥 oder ein Setup mit Prisma + MariaDB
* 🔥 oder ein Setup mit native MariaDB Pooling + Auto-Reconnect

Verwendest du Express, Fastify oder etwas anderes?
