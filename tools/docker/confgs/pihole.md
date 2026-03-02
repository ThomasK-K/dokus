
```
# Pi-hole mit Nginx Reverse Proxy (Docker Best Practice)

## Übersicht

```

FritzBox (DHCP + Gateway)
IP: 192.168.10.11
│
▼
Nginx (Reverse Proxy) ← Pi-hole Admin Interface
IP: 192.168.10.152
│
▼
Pi-hole (DNS Server)
│
▼
Upstream DNS (Quad9 / Cloudflare)

```

Ziel:
- FritzBox bleibt DHCP-Server
- Pi-hole ist **einziger DNS** für alle Clients
- Nginx als Reverse Proxy für sichere Admin-Seite
- Einfache Erreichbarkeit über Standard-Ports
- IPv4 & IPv6 sauber
- Stabil & AVM-kompatibel

---

## 1. Voraussetzungen

- FritzBox (IPv4 aktiv, IPv6 optional)
- Linux Host mit Docker & Docker Compose
- Statische IP für den Docker-Host:
```

192.168.10.152

````

---

## 2. Docker Compose Konfiguration mit Nginx

### Datei: `docker-compose.yml`

```yaml
version: "3.8"

networks:
  pihole_net:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/24

services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    hostname: pihole
    restart: unless-stopped
    networks:
      pihole_net:
        ipv4_address: 172.20.0.10
    ports:
      - "53:53/tcp"
      - "53:53/udp"
    environment:
      TZ: "Europe/Berlin"
      WEBPASSWORD: "HIER_SICHERES_PASSWORT"
      DHCP_ACTIVE: "false"
      PIHOLE_DNS_: "9.9.9.9;1.1.1.1"
      CACHE_SIZE: 10000
      BLOCKING_ENABLED: "true"
      # Interface auf alle setzen für Nginx Zugriff
      DNSMASQ_LISTENING: "all"
      WEB_PORT: "8080"
      # Conditional Forwarding zur FritzBox
      CONDITIONAL_FORWARDING: "true"
      CONDITIONAL_FORWARDING_IP: "192.168.10.11"
      CONDITIONAL_FORWARDING_DOMAIN: "fritz.box"
      CONDITIONAL_FORWARDING_REVERSE: "10.168.192.in-addr.arpa"
    volumes:
      - ./pihole/etc-pihole:/etc/pihole
      - ./pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN
    healthcheck:
      test: ["CMD", "dig", "google.com", "@127.0.0.1"]
      interval: 30s
      timeout: 5s
      retries: 3

  nginx:
    image: nginx:alpine
    container_name: nginx-pihole
    hostname: nginx-pihole
    restart: unless-stopped
    networks:
      pihole_net:
        ipv4_address: 172.20.0.20
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - pihole
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### Nginx Konfigurationsdateien

#### Datei: `nginx/nginx.conf`

```nginx
user nginx;
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;
    
    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript 
               text/xml application/xml application/xml+rss text/javascript;
    
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    
    include /etc/nginx/conf.d/*.conf;
}
```

#### Datei: `nginx/conf.d/pihole.conf`

```nginx
# Upstream für Pi-hole
upstream pihole {
    server 172.20.0.10:8080;
}

# HTTP to HTTPS Redirect
server {
    listen 80;
    server_name _;
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Redirect everything else to HTTPS
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS Server für Pi-hole
server {
    listen 443 ssl http2;
    server_name _;
    
    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/pihole.crt;
    ssl_certificate_key /etc/nginx/ssl/pihole.key;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Pi-hole Admin Interface
    location / {
        proxy_pass http://pihole;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket Support für Pi-hole
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    # Static files direkt von Pi-hole
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg)$ {
        proxy_pass http://pihole;
        proxy_cache_valid 200 1h;
        expires 1h;
        add_header Cache-Control "public, immutable";
    }
}
```

### SSL-Zertifikat erstellen

```bash
# Verzeichnis erstellen
mkdir -p nginx/ssl

# Self-signed Zertifikat für lokale Nutzung
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout nginx/ssl/pihole.key \
    -out nginx/ssl/pihole.crt \
    -subj "/C=DE/ST=YourState/L=YourCity/O=HomeNetwork/CN=pihole.fritz.box"

# Berechtigungen setzen
chmod 600 nginx/ssl/pihole.key
chmod 644 nginx/ssl/pihole.crt
```

### Warum Bridge-Netzwerk statt Host?

* Bessere Isolation zwischen Services
* Nginx kann als Reverse Proxy fungieren
* SSL/TLS Terminierung möglich
* Flexiblere Port-Konfiguration
* Bessere Sicherheit durch Netzwerk-Segmentierung

---

## 3. FritzBox Konfiguration

### 3.1 DHCP Server

```
Heimnetz → Netzwerk → Netzwerkeinstellungen

✔ DHCP-Server aktiv
IP-Bereich: 192.168.10.20 – 192.168.10.200

Lease-Zeit: 1 Woche
```

---

### 3.2 DNS an LAN-Clients

```
DNS-Server an LAN-Geräte weiterreichen: ✔

Primärer DNSv4: 192.168.10.152
Sekundärer DNSv4: (leer)
```

➡ Clients können Pi-hole nicht umgehen

---

### 3.3 FritzBox selbst (wichtig!)

```
Internet → Filter → Listen

DNS vom Anbieter verwenden: ❌
Andere DNS verwenden: ✔

Primärer DNSv4: 192.168.10.152
Sekundärer DNSv4: 9.9.9.9
```

➡ FritzBox bleibt erreichbar bei Pi-hole-Neustart

---

### 3.4 IPv6 (empfohlen)

**Option A – Sauber**

```
Primärer DNSv6: Pi-hole IPv6
Sekundärer DNSv6: leer
```

**Option B – IPv6 DNS deaktivieren**

```
Internet → Zugangsdaten → IPv6
DNSv6 verteilen: ❌
```

---

## 4. Pi-hole Web Interface Einstellungen

### Settings → DNS

* ✔ Quad9 + Cloudflare
* ✔ Use DNSSEC
* ✔ Never forward non-FQDNs
* ✔ Never forward reverse lookups

---

### Settings → Advanced

```
Interface listening behavior:
→ Listen on all interfaces
```

---

## 5. Lokale DNS-Einträge (optional)

```bash
echo "192.168.10.11 fritz.box fritz" >> ./etc-pihole/custom.list
echo "192.168.10.152 pihole.fritz.box pihole" >> ./etc-pihole/custom.list
docker restart pihole
```

---

## 6. Setup und Start

### Verzeichnisstruktur erstellen

```bash
# Hauptverzeichnis erstellen
mkdir -p pihole-nginx
cd pihole-nginx

# Unterverzeichnisse
mkdir -p nginx/{conf.d,ssl} pihole/{etc-pihole,etc-dnsmasq.d}

# SSL-Zertifikat erstellen
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout nginx/ssl/pihole.key \
    -out nginx/ssl/pihole.crt \
    -subj "/C=DE/ST=Germany/L=Home/O=HomeNetwork/CN=pihole.fritz.box"

chmod 600 nginx/ssl/pihole.key
chmod 644 nginx/ssl/pihole.crt
```

### Nginx-Konfiguration erstellen

```bash
# nginx.conf erstellen (siehe oben)
# nginx/conf.d/pihole.conf erstellen (siehe oben)
```

### Docker Services starten

```bash
docker compose up -d
docker compose logs -f
```

### Web UI Zugriff

```
# HTTP (wird automatisch zu HTTPS weitergeleitet)
http://192.168.10.152

# HTTPS (direkt)
https://192.168.10.152

# Lokaler DNS-Name (nach DNS-Konfiguration)
https://pihole.fritz.box
```

---

## 7. Tests & Validierung

### Client

```bash
ipconfig /all      # Windows
resolvectl status  # Linux
```

➡ DNS: `192.168.10.152`

---

### Pi-hole

```bash
dig google.com @127.0.0.1
dig fritz.box
```

---

## 8. Vorteile der Nginx + Pi-hole Konfiguration

### ✅ Sicherheit
- SSL/TLS-Verschlüsselung für Admin-Interface
- Security Headers (XSS, CSRF, etc.)
- Reverse Proxy isoliert Pi-hole vom direkten Zugriff
- Möglichkeit für Basic Auth oder andere Authentifizierung

### ✅ Zugänglichkeit
- Standard-Ports (80/443) statt Pi-hole's 80/admin
- Automatische HTTP-zu-HTTPS-Weiterleitung
- Saubere URLs ohne `/admin` Pfad
- Lokale DNS-Namen möglich

### ✅ Wartung & Monitoring
- Getrennte Container für bessere Updates
- Health Checks für beide Services
- Bessere Log-Trennung
- Einfachere Backup-Strategien

### ✅ Erweiterbarkeit
- Weitere Services können hinter Nginx
- Load Balancing für mehrere Pi-hole Instanzen
- Rate Limiting möglich
- Caching für statische Inhalte

---

## 9. Häufige Fehler (vermieden)

| Fehler                | Status |
| --------------------- | ------ |
| Unverschlüsseltes Web UI | ❌      |
| Direkte Pi-hole Exposition | ❌      |
| Port-Konflikte        | ❌      |
| Fehlende Security Headers | ❌      |
| Komplexe Netzwerk-Setup | ❌      |

---

## 9. Optional: Erweiterungen

* Unbound als lokaler Resolver
* DNS-over-HTTPS
## 10. Optional: Erweiterungen

### Authentifizierung hinzufügen

```nginx
# In nginx/conf.d/pihole.conf ergänzen
location / {
    auth_basic "Pi-hole Admin";
    auth_basic_user_file /etc/nginx/.htpasswd;
    # ... restliche Konfiguration
}
```

### Let's Encrypt SSL (für öffentliche Domains)

```yaml
# Docker Compose ergänzen
  certbot:
    image: certbot/certbot
    volumes:
      - ./nginx/ssl:/etc/letsencrypt
    command: certonly --webroot -w /var/www/html -d pihole.yourdomain.com
```

### Monitoring mit Prometheus

```yaml
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

---

## Fazit

Diese Konfiguration bietet:

### 🔒 Sicherheit
- SSL/TLS-verschlüsselte Admin-Oberfläche
- Security Headers gegen XSS und andere Angriffe
- Isolation durch Reverse Proxy

### 🌐 Zugänglichkeit
- Standard-Ports (80/443) ohne komplexe URLs
- Automatische HTTP-zu-HTTPS-Weiterleitung
- Saubere, professionelle Lösung

### ⚡ Performance & Wartung
- Health Checks für beide Services
- Getrennte Container für bessere Updates
- Gzip-Kompression für statische Inhalte
- Caching-Optimierungen

### 📈 Erweiterbarkeit
- Bereit für zusätzliche Services
- Load Balancing möglich
- Monitoring und Authentifizierung einfach erweiterbar

**Empfohlen für produktive Heim- und SOHO-Netze mit professionellen Anforderungen.**

```

---

Wenn du willst, kann ich dir als Nächstes:

- 📄 **zweite Markdown-Datei für Unbound**
- 🔐 **DoH + DNSSEC Hardening**
- 🔁 **Redundanz mit 2 Pi-holes**
- 📦 **Git-Repo-Struktur (README + env.example)**

Sag einfach Bescheid.
```