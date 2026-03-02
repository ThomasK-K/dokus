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

ls -l /etc/resolv.conf


Erwartet:

/etc/resolv.conf -> /run/systemd/resolve/resolv.conf

4. DNS-Einträge dauerhaft ändern

Erstelle die Konfiguration für systemd-resolved:

sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/dns.conf <<EOF
[Resolve]
DNS=8.8.8.8 1.1.1.1
FallbackDNS=9.9.9.9
EOF


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

7. Zusammenfassung

LAN prüfen (Ping auf IP)

systemd-resolved aktivieren

/etc/resolv.conf korrekt verlinken

DNS in /etc/systemd/resolved.conf.d/dns.conf setzen

NetworkManager Verbindung ggf. anpassen

Dienste neu starten und testen