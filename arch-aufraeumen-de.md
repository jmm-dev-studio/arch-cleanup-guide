# Arch Linux schlank halten: Sicheres Aufräumen von Cache, Waisen und Logs

*Von JMM Dev Studio*

Arch Linux räumt nicht hinter sich auf. Das ist Absicht: pacman behält alte Paketversionen, damit man ein Downgrade machen kann, und systemd behält Logs für die Fehlersuche. Nach einem Jahr Updates können diese Reste jedoch viele Gigabyte belegen. Dieser Artikel zeigt eine kurze, sichere Routine, die Sie einmal im Monat ausführen können. Jeder Befehl wird erklärt, und nichts davon löscht Ihre persönlichen Dateien.

## 1. Erst messen, wo der Platz bleibt

Bevor Sie etwas löschen, schauen Sie nach. Beginnen Sie mit Paketcache und Journal:

```
du -sh /var/cache/pacman/pkg
journalctl --disk-usage
```

Bei einem System, das länger nicht aufgeräumt wurde, liegt die erste Zahl oft bei 5 bis 30 GB.

Für eine interaktive Ansicht der ganzen Platte installieren Sie `ncdu` und starten es auf dem Root-Dateisystem, ohne in andere Mounts zu wechseln:

```
sudo pacman -S ncdu
sudo ncdu -x /
```

## 2. pacman-Cache ausdünnen

Jedes Paket, das Sie je installiert oder aktualisiert haben, liegt noch in `/var/cache/pacman/pkg`. Das einfachste Werkzeug ist `paccache` aus dem Paket `pacman-contrib`:

```
sudo pacman -S pacman-contrib
paccache -d
```

Die Option `-d` ist ein Probelauf: Es wird nur angezeigt, was entfernt würde. Standardmäßig behält `paccache` die drei neuesten Versionen jedes Pakets. Wenn die Liste passt, führen Sie es richtig aus:

```
sudo paccache -r
```

Wenn Sie weniger Versionen behalten möchten, nutzen Sie `-k`:

```
sudo paccache -rk1
```

Den Cache von Paketen, die nicht mehr installiert sind, entfernen Sie so:

```
sudo paccache -ruk0
```

Vermeiden Sie `pacman -Scc`, außer Sie haben wenig Platz und kennen den Preis. Der Befehl löscht den gesamten Cache. Dann können Sie ein fehlerhaftes Update nicht mehr zurückrollen, ohne das alte Paket erneut herunterzuladen.

Optional erledigt systemd das jede Woche für Sie:

```
sudo systemctl enable --now paccache.timer
```

## 3. Verwaiste Pakete entfernen

Waisen sind Pakete, die als Abhängigkeit installiert wurden, aber von nichts mehr gebraucht werden. Lassen Sie sich zuerst die Liste anzeigen:

```
pacman -Qtdq
```

Ist die Liste leer, gibt pacman nichts aus und meldet einen Fehlercode. Das ist normal. Lesen Sie die Liste genau. Erkennen Sie ein Paket, das Sie behalten wollen, markieren Sie es als explizit installiert:

```
sudo pacman -D --asexplicit paketname
```

Wenn die Liste in Ordnung ist, entfernen Sie die Waisen samt Konfigurationsdateien:

```
sudo pacman -Rns $(pacman -Qtdq)
```

In der Fish-Shell schreiben Sie dasselbe als `sudo pacman -Rns (pacman -Qtdq)`.

## 4. systemd-Journal begrenzen

Das Journal kann auf mehrere Gigabyte wachsen. Prüfen Sie die Größe und kürzen Sie es auf ein vernünftiges Maß:

```
sudo journalctl --vacuum-size=200M
```

Alternativ nach Alter:

```
sudo journalctl --vacuum-time=4weeks
```

Damit die Grenze dauerhaft gilt, legen Sie eine Drop-in-Datei an, statt die Hauptkonfiguration zu bearbeiten:

```
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=200M\n' | sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald
```

## 5. AUR-Build-Dateien und Caches prüfen

Wenn Sie einen AUR-Helper wie `yay` nutzen, liegen geklonte Build-Verzeichnisse unter `~/.cache/yay`. Sie können sie gefahrlos löschen, der Helper klont sie bei Bedarf neu:

```
du -sh ~/.cache/yay
yay -Sc
```

Lesen Sie die Rückfragen von `yay -Sc`, bevor Sie bestätigen. Für andere Daten in `~/.cache` schauen Sie sich zuerst die größten Ordner an:

```
du -sh ~/.cache/* | sort -h | tail
```

Löschen Sie nur Caches von Programmen, die Sie kennen. Löschen Sie niemals `~/.config` oder `~/.local/share` im Rahmen einer Aufräumaktion.

## 6. Fehlgeschlagene Dienste und Konfigurationsreste prüfen

Beim Aufräumen bietet sich ein kurzer Gesundheitscheck an:

```
systemctl --failed
sudo find /etc -name "*.pacnew" -o -name "*.pacsave"
```

Eine `.pacnew`-Datei bedeutet: Ein Paket brachte eine neue Standardkonfiguration mit, und pacman hat Ihre nicht überschrieben. Vergleichen Sie beide mit `diff` oder mit `pacdiff` aus `pacman-contrib`, übernehmen Sie, was Sie brauchen, und löschen Sie die Reste.

## Eine Monatsroutine in fünf Zeilen

```
sudo paccache -rk2
sudo pacman -Rns $(pacman -Qtdq)
sudo journalctl --vacuum-size=200M
systemctl --failed
sudo find /etc -name "*.pacnew"
```

Führen Sie die Zeile für die Waisen erst aus, nachdem Sie die Liste gelesen haben. Automatisieren ist für Cache und Journal sinnvoll, bei der Paketentfernung sollte ein Mensch hinsehen.

## Zusammenfassung

- Erst messen mit `du`, `journalctl --disk-usage` und `ncdu`.
- `paccache` ist sicherer als `pacman -Scc`, weil es aktuelle Versionen für Downgrades behält.
- Waisen mit `pacman -Qtdq` prüfen, bevor Sie sie entfernen.
- Das Journal per Drop-in-Datei begrenzen, damit die Grenze Neustarts übersteht.
- `~/.config` und `~/.local/share` nie blind aufräumen.

Mit dieser Routine bleibt ein typisches Arch-System klein und berechenbar, und Ihr Sicherheitsnetz für Rollbacks bleibt erhalten.
