Perfekt 👍 dann reden wir über **`docker compose` ohne Swarm** in der
**Docker Community Edition**.

Da gibt es **keine echten Docker-Secrets**, deshalb ist die sauberste Lösung:

> ✅ **Secrets als Datei mounten (Best Practice für Compose)**

---

# 🔐 Empfohlene Lösung für `docker compose`

## 1️⃣ Secret-Datei anlegen

Projektstruktur:

```
project/
 ├─ docker-compose.yml
 └─ secrets/
     └─ db_password.txt
```

Datei erstellen:

```bash
mkdir -p secrets
echo "SuperSicheresPasswort123" > secrets/db_password.txt
chmod 600 secrets/db_password.txt
```

Wichtig:

* `.gitignore` → `secrets/`
* Nur dein User sollte Zugriff haben (`chmod 600`)

---

## 2️⃣ docker-compose.yml

```yaml
services:
  mariadb:
    image: mariadb:11
    environment:
      MARIADB_ROOT_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - ./secrets/db_password.txt:/run/secrets/db_password:ro
    restart: unless-stopped

  api:
    build: .
    volumes:
      - ./secrets/db_password.txt:/run/secrets/db_password:ro
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
    depends_on:
      - mariadb
    restart: unless-stopped
```

---

## 3️⃣ Passwort im Code lesen (Node.js Beispiel)

```js
import fs from "fs";

const password = fs.readFileSync(
  process.env.DB_PASSWORD_FILE,
  "utf8"
).trim();
```

---

# 🔥 Warum das besser ist als `.env`?

| Methode              | Sicher?  | Sichtbar via `docker inspect`? |
| -------------------- | -------- | ------------------------------ |
| `.env`               | ❌ mittel | ✅ Ja                           |
| Environment Variable | ❌ mittel | ✅ Ja                           |
| Datei-Mount          | ✅ besser | ❌ Nein                         |

Das Secret:

* liegt nicht im Compose-File
* ist nicht im Container-Environment
* ist read-only gemountet

---

# 🧠 Extra-Tipp für Ubuntu 24 (dein Setup)

Da du **Ubuntu 24** nutzt:

* Speichere Secrets z.B. unter:

  ```
  /opt/myapp/secrets/
  ```
* Setze Owner auf root:

  ```bash
  sudo chown root:root db_password.txt
  sudo chmod 600 db_password.txt
  ```

Dann im Compose:

```yaml
- /opt/myapp/secrets/db_password.txt:/run/secrets/db_password:ro
```

Noch sicherer 👍

---

# 🚀 Wenn du es richtig professionell willst

Dann nimm z.B.:

* HashiCorp Vault
* oder später Kubernetes Secrets

Aber für dein Setup (Backend + MariaDB + Compose)
ist Datei-Mount absolut ausreichend.

---

Wenn du willst, baue ich dir:

* 🔒 ein komplett sicheres Compose-Template (API + MariaDB + Reverse Proxy)
* 🛡️ mit separatem Netzwerk + Non-Root Container
* 🔄 mit automatischem Backup

Wie betreibst du es aktuell – nur lokal oder auf einem Server?
