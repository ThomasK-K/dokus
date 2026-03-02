# Ubuntu 24.04 LAN & DNS Konfiguration

Dieses Dokument beschreibt, wie man LAN (Ethernet) unter Ubuntu 24.04 korrekt konfiguriert, systemd-resolved nutzt und DNS-Einträge dauerhaft ändert.

---

## 1. Prüfen der Netzwerkverbindung

```bash
ip a           # Zeigt Netzwerk-Interfaces
nmcli device    # Zeigt Status der Netzwerkgeräte
ping -c 3 8.8.8.8  # Testet Verbindung zum Internet über IP


✅ Wenn der Ping auf 8.8.8.8 funktioniert, ist das Netzwerk aktiv und Routing korrekt.

2. Prüfen von systemd-resolved
systemctl status systemd-resolved


active (running) → DNS-Dienst läuft

Falls nicht, aktivieren:

sudo systemctl enable systemd-resolved
sudo systemctl start systemd-resolved

3. /etc/resolv.conf korrekt setzen

Ubuntu 24.04 erwartet einen Symlink auf die von systemd-resolved verwaltete Datei.

sudo chattr -i /etc/resolv.conf 2>/dev/null
sudo rm -f /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf


Prüfen:

```bash
ls -l /etc/resolv.conf
```

Erwartet:

```
/etc/resolv.conf -> /run/systemd/resolve/resolv.conf
```

**Inhalt von resolv.conf überprüfen:**

```bash
cat /etc/resolv.conf
```



❌ **Fehlerhaft:** `nameserver 192.168.10.11 8.8.8.8 1.1.1.1` (mehrere IPs in einer Zeile)  
✅ **Korrekt:** Jeder nameserver in separater Zeile

4. DNS-Einträge dauerhaft ändern

Erstelle die Konfiguration für systemd-resolved:

```
# This is /run/systemd/resolve/resolv.conf managed by man:systemd-resolved(8).
nameserver 192.168.10.11
nameserver 8.8.8.8
nameserver 1.1.1.1
search fritz.box
```


DNS= → primäre DNS-Server

FallbackDNS= → alternative DNS-Server, falls primäre ausfallen

Dienste neu starten:

sudo systemctl restart systemd-resolved
sudo systemctl restart NetworkManager


Prüfen:

resolvectl status
getent hosts google.com
ping -c 3 google.com

5. NetworkManager DNS-Einstellungen (optional)

Manuelles Setzen der DNS-Server für eine Verbindung:

nmcli connection show            # Liste der Verbindungen
nmcli connection modify "Wired connection 1" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes
nmcli connection down "Wired connection 1"
nmcli connection up "Wired connection 1"


ipv4.ignore-auto-dns yes → verhindert, dass DHCP DNS überschreibt

Die Änderung bleibt dauerhaft bestehen

6. Hinweise

Nie dauerhaft /etc/resolv.conf manuell editieren, wenn systemd-resolved aktiv ist

VPN oder Docker können DNS überschreiben; ggf. separat konfigurieren

DNS regelmäßig testen:

getent hosts de.archive.ubuntu.com
ping -c 3 google.com

## 7. Problembehebung

**Falls nameserver-Einträge falsch formatiert sind:**

```bash
# Dienste neu starten um korrekte Formatierung zu erzwingen
sudo systemctl restart systemd-resolved
sudo systemctl restart NetworkManager

# Danach erneut prüfen
cat /etc/resolv.conf
```

**Bei "Too many DNS servers" Warnung:**
- Maximum 3 nameserver-Einträge werden unterstützt
- Reduziere DNS-Server in `/etc/systemd/resolved.conf.d/dns.conf`

**Bei "ping google.de temporärer Fehler" (DNS-Auflösungsfehler):**

```bash
# 1. DNS-Cache leeren (Ubuntu 24.04)
sudo resolvectl flush-caches
# Alternativ auf älteren Systemen: sudo systemd-resolve --flush-caches

# 2. DNS-Status prüfen
resolvectl status

# 3. DNS-Auflösung testen
nslookup google.de
dig google.de

# 4. Alternative DNS-Server temporär testen
nslookup google.de 8.8.8.8

# 5. Falls DNS nicht funktioniert, NetworkManager neu starten
sudo systemctl restart NetworkManager
sudo systemctl restart systemd-resolved
```

**Bei "Ping geht, aber kein DNS" (IP funktioniert, Hostnamen nicht):**

```bash
# 1. Problem bestätigen
ping -c 1 8.8.8.8        # ✅ sollte funktionieren
ping -c 1 google.de      # ❌ schlägt fehl

# 2. DNS-Server überprüfen
cat /etc/resolv.conf      # Sind DNS-Server eingetragen?
resolvectl dns            # Welche DNS-Server sind aktiv?

# 3. DNS manuell testen
nslookup google.de
# Falls "connection timed out" → DNS-Server nicht erreichbar

# 4. DNS-Server neu setzen
sudo resolvectl dns enp0s3 8.8.8.8 1.1.1.1  # Interface anpassen!
# oder über NetworkManager:
nmcli connection modify "Wired connection 1" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection down "Wired connection 1" && nmcli connection up "Wired connection 1"

# 5. Testen
ping -c 1 google.de
```

**✅ Falls DNS jetzt funktioniert - Dauerhaft machen:**

```bash
# Temporäre Lösung mit resolvectl ist nur bis zum Neustart!
# Für dauerhafte Lösung eine der folgenden Methoden verwenden:

# Methode 1: NetworkManager (empfohlen)
nmcli connection show  # Verbindungsname notieren
nmcli connection modify "Wired connection 1" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli connection modify "Wired connection 1" ipv4.ignore-auto-dns yes
nmcli connection down "Wired connection 1" && nmcli connection up "Wired connection 1"

# Methode 2: systemd-resolved Konfiguration
sudo tee /etc/systemd/resolved.conf.d/dns.conf <<EOF
[Resolve]
DNS=8.8.8.8 1.1.1.1
FallbackDNS=9.9.9.9
EOF
sudo systemctl restart systemd-resolved

# Prüfen dass es dauerhaft funktioniert
ping -c 1 google.de
sudo reboot  # Nach Neustart erneut testen
```

**Diagnostik:**
```bash
# Prüfe welche DNS-Server aktiv sind
resolvectl dns

# Teste verschiedene Domains
ping -c 1 8.8.8.8      # IP funktioniert?
ping -c 1 google.com   # DNS funktioniert?
ping -c 1 google.de    # Spezifische Domain
```

## 8. Zusammenfassung

LAN prüfen (Ping auf IP)

systemd-resolved aktivieren

/etc/resolv.conf korrekt verlinken

DNS in /run/systemd/resolve/resolved.conf

NetworkManager Verbindung ggf. anpassen

Dienste neu starten und testen