## 3. FritzBox Konfiguration (Multi-Service Setup)

### 3.1 DHCP Server
```
Heimnetz → Netzwerk → Netzwerkeinstellungen

✔ DHCP-Server aktiv
IP-Bereich: 192.168.10.20 – 192.168.10.200
Gateway: 192.168.10.11 (FritzBox IP)
```

### 3.2 DNS an LAN-Clients
```
DNS-Server an LAN-Geräte weiterreichen: ✔

Primärer DNSv4: 192.168.10.152 (Docker Host)
Sekundärer DNSv4: (leer lassen!)
```

### 3.3 IPv6 deaktivieren (empfohlen für einfache Setups)
```
Internet → Zugangsdaten → IPv6
IPv6-Unterstützung: ❌
DNSv6 verteilen: ❌
```

```
Heimnetz → Netzwerk → Netzwerkeinstellungen
IPv6-Adressen zuweisen: ❌
```