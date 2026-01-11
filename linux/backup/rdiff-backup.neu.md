# rdiff-backup Kommando-Referenz (Version 2.2.6 - Neue API)
# =========================================================

# Für deinen spezifischen Fehler - Korrekte Befehle:

# 1. Repository mit neuer API initialisieren/überprüfen
rdiff-backup --new list increments /media/tkk/backup_daten

# 2. Mit API-Version für Kompatibilität
rdiff-backup --api-version 201 list increments /media/tkk/backup_daten

# Grundlegende neue Syntax
# ------------------------

# Backup erstellen (neue Syntax)
rdiff-backup --new backup /quellverzeichnis /zielverzeichnis

# Increments auflisten (korrekt für 2.2.6)
rdiff-backup --new list increments /backup-ziel
rdiff-backup --api-version 201 list increments /backup-ziel

# Wiederherstellung (neue Syntax)
rdiff-backup --new restore --at 1D /backup-ziel /wiederherstellungs-ziel
rdiff-backup --api-version 201 restore --at 1D /backup-ziel /wiederherstellungs-ziel

# Backup bereinigen
rdiff-backup --new delete --older-than 30D /backup-ziel
rdiff-backup --api-version 201 delete --older-than 30D /backup-ziel

# Komplette Befehlsreferenz für 2.2.6
# ------------------------------------

# Hilfe anzeigen (neue API)
rdiff-backup --new --help
rdiff-backup --api-version 201 --help

# Backup Operationen
rdiff-backup --new backup /quelle /ziel
rdiff-backup --new backup --exclude "*.tmp" /quelle /ziel
rdiff-backup --api-version 201 backup /quelle /ziel

# Increments verwalten
rdiff-backup --new list increments /backup-ziel
rdiff-backup --new list increments --format JSON /backup-ziel
rdiff-backup --api-version 201 list increments /backup-ziel

# Wiederherstellung
rdiff-backup --new restore --at now /backup-ziel /wiederherstellungs-ziel
rdiff-backup --new restore --at 1D /backup-ziel /wiederherstellungs-ziel
rdiff-backup --new restore --at 2024-01-15 /backup-ziel /wiederherstellungs-ziel
rdiff-backup --api-version 201 restore --at 1D /backup-ziel /wiederherstellungs-ziel

# Bestimmte Datei wiederherstellen
rdiff-backup --new restore --at 2D /backup-ziel/pfad/datei /ziel-pfad

# Backup löschen/bereinigen
rdiff-backup --new delete --older-than 7D /backup-ziel
rdiff-backup --new delete --older-than 1M /backup-ziel
rdiff-backup --new delete --force --older-than 30D /backup-ziel
rdiff-backup --api-version 201 delete --older-than 30D /backup-ziel

# Repository Informationen
rdiff-backup --new info /backup-ziel
rdiff-backup --new info --statistics /backup-ziel
rdiff-backup --api-version 201 info /backup-ziel

# Vergleich
rdiff-backup --new compare /quelle /backup-ziel
rdiff-backup --new compare --at 1D /quelle /backup-ziel

# Remote Backup (neue Syntax)
rdiff-backup --new backup /lokale/quelle user@remotehost::/remote/backup
rdiff-backup --api-version 201 backup /lokale/quelle user@remotehost::/remote/backup

# Für deinen speziellen Fall - Lösungsansätze:
# --------------------------------------------

# 1. Prüfen ob Repository existiert
rdiff-backup --new info /media/tkk/backup_daten

# 2. Falls kein Repository existiert, erstelle eines:
rdiff-backup --new backup /deine/daten /media/tkk/backup_daten

# 3. Oder mit API-Version für Kompatibilität
rdiff-backup --api-version 201 backup /deine/daten /media/tkk/backup_daten

# 4. Altes Repository migrieren (falls vorhanden)
rdiff-backup --new migrate /media/tkk/backup_daten

# Praktische Beispiele für 2.2.6
# ------------------------------

# Backup mit Ausschlüssen
rdiff-backup --new backup \
  --exclude "*.tmp" \
  --exclude "**/cache/" \
  /home/user \
  /media/tkk/backup_daten

# Increments anzeigen (dein spezifischer Fall)
rdiff-backup --new list increments /media/tkk/backup_daten
rdiff-backup --api-version 201 list increments /media/tkk/backup_daten

# Automatisierung mit Cron (neue Syntax)
# 0 2 * * * rdiff-backup --api-version 201 backup /important /media/tkk/backup_daten
# 0 3 * * 0 rdiff-backup --api-version 201 delete --older-than 30D /media/tkk/backup_daten

# Fehlerbehebung für deinen Fall:
# -------------------------------

# 1. Repository-Struktur prüfen
ls -la /media/tkk/backup_daten/

# 2. Falls rdiff-backup-data Ordner existiert, ist es ein altes Repository
# 3. Dann migrieren oder mit API-Version arbeiten

# Migration von altem Repository
rdiff-backup --new migrate /media/tkk/backup_daten

# Oder mit Kompatibilitätsmodus arbeiten
rdiff-backup --api-version 201 list increments /media/tkk/backup_daten

# Wichtig in Version 2.2.6:
# - Alte Syntax ist deprecated
# --new für neue API verwenden
# --api-version für Kompatibilität
# Repository muss mit neuer API erstellt werden