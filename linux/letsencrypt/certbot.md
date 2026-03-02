Hier ist ein **komplettes Setup für Ubuntu 24 + Nginx + Certbot + certbot-dns-multi mit IPv64**.
Ich mache es so, dass du es 1:1 ausführen kannst.

---

# 🧰 Ziel

* Ubuntu 24
* Nginx
* Wildcard-Zertifikat (`*.example.ipv64.net`)
* DNS-01-Challenge über IPv64 API
* Auto-Renew per systemd

---

# 1️⃣ Pakete installieren

```bash
sudo apt update
sudo apt install -y nginx python3 python3-venv python3-pip
```

---

# 2️⃣ Certbot + certbot-dns-multi installieren

Wir nutzen eine virtuelle Umgebung (sauberer als System-Python):

```bash
sudo mkdir -p /opt/certbot
sudo python3 -m venv /opt/certbot/venv
sudo /opt/certbot/venv/bin/pip install --upgrade pip
sudo /opt/certbot/venv/bin/pip install certbot certbot-dns-multi
```

Optional Symlink:

```bash
sudo ln -s /opt/certbot/venv/bin/certbot /usr/local/bin/certbot
```

Test:

```bash
certbot --version
```

---

# 3️⃣ IPv64 API Key vorbereiten

Im Dashboard von IPv64 deinen API-Key erzeugen.

Datei erstellen:

```bash
sudo nano /root/ipv64.ini
```

Inhalt:

```ini
dns_ipv64_api_key = DEIN_API_KEY
```

Rechte setzen (wichtig!):

```bash
sudo chmod 600 /root/ipv64.ini
```

---

# 4️⃣ Zertifikat erstellen (inkl. Wildcard)

Beispiel für:

```
example.ipv64.net
*.example.ipv64.net
```

```bash
sudo certbot certonly \
  --authenticator dns-multi \
  --dns-multi-credentials /root/ipv64.ini \
  --dns-multi-providers ipv64 \
  -d example.ipv64.net \
  -d '*.example.ipv64.net'
```

Wenn alles klappt, findest du die Zertifikate hier:

```
/etc/letsencrypt/live/example.ipv64.net/
```

---

# 5️⃣ Nginx konfigurieren

```bash
sudo nano /etc/nginx/sites-available/example
```

Beispiel:

```nginx
server {
    listen 443 ssl;
    server_name example.ipv64.net *.example.ipv64.net;

    ssl_certificate /etc/letsencrypt/live/example.ipv64.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.ipv64.net/privkey.pem;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

Aktivieren:

```bash
sudo ln -s /etc/nginx/sites-available/example /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

# 6️⃣ Auto-Renew testen

Certbot bringt bereits einen systemd-Timer mit.

Testlauf:

```bash
sudo certbot renew --dry-run
```

Wenn das funktioniert → alles korrekt eingerichtet.

---

# 7️⃣ Optional: Hook für Nginx Reload

Falls nötig:

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Inhalt:

```bash
#!/bin/bash
systemctl reload nginx
```

Rechte:

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

---

# 🔐 Ergebnis

* DNS-Challenge läuft über IPv64 API
* Kein Port 80 nötig
* Wildcard funktioniert
* Automatische Verlängerung
* Nginx wird nach Erneuerung neu geladen

---

Wenn du willst, kann ich dir noch zeigen:

* 🔄 Setup für mehrere Domains bei verschiedenen DNS-Providern
* 🐳 Variante als Docker-Container
* 🔁 Integration in ein CI/CD-Deployment
* 🔐 Reverse-Proxy Setup mit mehreren Services

Was ist dein konkretes Ziel?
