Reolink-Kameras sind bekannt dafür, dass sie bei der Standard-Einbindung in Frigate manchmal Bildartefakte ("Smearing") oder graue Bilder zeigen, weil ihre RTSP-Implementierung etwas eigenwillig ist.

Die Lösung ist die Nutzung von **Go2RTC** (ist in aktuellen Frigate-Versionen integriert) und idealerweise des **HTTP-FLV Streams** anstelle von RTSP. Das läuft deutlich stabiler.

Hier ist eine bewährte `config.yml` für eine Reolink-Kamera (z.B. RLC-510A, 520A, 810A etc.).

### Deine `frigate/config/config.yml`

```yaml
mqtt:
  host: 192.168.1.100  # IP deines MQTT Brokers (oft derselbe Server)
  # user: dein_user    # Optional
  # password: dein_pw  # Optional

# Go2RTC ist die "Zauber-Komponente". 
# Sie holt den Stream effizient von der Kamera und verteilt ihn an Frigate und Home Assistant.
go2rtc:
  streams:
    # Wir benennen die Streams logisch
    # "main" = Hohe Qualität für Aufnahme
    # "sub" = Niedrige Qualität für Erkennung (spart CPU)
    
    # TRICK: Wir nutzen http-flv statt rtsp für Reolink (stabiler!)
    reolink_garten_main:
      - "ffmpeg:http://192.168.1.50/flv?port=1935&app=bcs&stream=channel0_main.bcs&user=admin&password=DEIN_PASSWORT#video=copy#audio=copy#audio=aac"
    
    reolink_garten_sub:
      - "ffmpeg:http://192.168.1.50/flv?port=1935&app=bcs&stream=channel0_ext.bcs&user=admin&password=DEIN_PASSWORT"

cameras:
  garten_kamera: # Der Name der Kamera in Frigate
    ffmpeg:
      inputs:
        # Wir greifen hier auf die oben definierten Go2RTC Streams zu
        # INPUT 1: High Res für Aufnahmen
        - path: rtsp://127.0.0.1:8554/reolink_garten_main
          roles:
            - record
        
        # INPUT 2: Low Res für Erkennung (Detect)
        - path: rtsp://127.0.0.1:8554/reolink_garten_sub
          roles:
            - detect
            
    # Auflösung des SUB-Streams (Wichtig! Muss exakt stimmen!)
    # Reolink Sub-Streams sind oft 640x360 oder 640x480. 
    # Prüfe das im Reolink Client unter "Stream-Einstellungen" -> "Flüssig".
    detect:
      width: 640
      height: 360 
      fps: 5 # Frigate braucht für Erkennung nicht mehr als 5 FPS

    # Aktivieren der Aufzeichnung
    record:
      enabled: True
      retain:
        days: 3
        mode: motion # Nur aufnehmen, wenn Bewegung erkannt wurde
      events:
        retain:
          default: 10 # Wichtige Events (z.B. Person) 10 Tage behalten
          mode: active_objects

    # Schnappschüsse für Home Assistant Benachrichtigungen
    snapshots:
      enabled: True
      timestamp: True
      bounding_box: True

    # Hier definierst du, wonach gesucht werden soll
    objects:
      track:
        - person
        - dog
        - cat
        - car

```

### Die 3 wichtigsten Fallstricke bei Reolink & Frigate

Damit diese Config funktioniert, musst du drei Dinge beachten:

1. **IP-Adresse & Passwort:** Ersetze `192.168.1.50` und `DEIN_PASSWORT` in der `go2rtc` Sektion.
2. **Ports aktivieren:** Wie vorhin erwähnt, muss in der Reolink-Software (Client am PC/Mac) unter *Netzwerk -> Erweitert -> Server-Einstellungen* der **RTMP** und **HTTP** Port aktiviert sein. Sonst funktioniert der Link oben nicht.
3. **H.265 Problem (bei 4K/8MP Kameras):**
* Moderne 4K Reolink Kameras nutzen für den Main-Stream **H.265**.
* Die meisten Browser (Chrome/Firefox) können H.265 **nicht** nativ abspielen.
* *Konsequenz:* In Home Assistant siehst du dann oft kein Bild beim Main-Stream.
* *Lösung:* Wenn du H.265 Kameras hast, musst du Go2RTC anweisen, den Stream für den Browser umzuwandeln (transcoding), was CPU kostet, oder du lebst damit, dass du live meistens nur den Sub-Stream anschaust.



### Hardware-Frage

Frigate benötigt sehr viel CPU-Leistung für die Objekterkennung, wenn man keinen Beschleuniger hat.

**Hast du bereits einen Google Coral USB Stick (TPU) eingeplant oder willst du die Erkennung erst einmal über die normale CPU laufen lassen?** (Das beeinflusst, wie wir die `detectors` Sektion konfigurieren müssen).