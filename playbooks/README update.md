# Ubuntu-Update-Playbook

## Zweck

Dieses Ansible-Playbook aktualisiert die installierten Ubuntu-Pakete auf allen Hosts des Inventars. Vor und nach dem Update werden wichtige Systeminformationen ermittelt. Anschließend wird ein zentraler Update-Report erstellt und per E-Mail versendet.

Das Playbook führt aktuell ein normales Paket-Upgrade durch. Ein vollständiges Distribution-Upgrade, ein automatischer Neustart und das Entfernen nicht benötigter Pakete sind derzeit deaktiviert.

---

## Voraussetzungen

Für die Ausführung werden folgende Voraussetzungen benötigt:

- Ansible auf dem Control-Host
- Erreichbare Ubuntu-Zielsysteme
- SSH-Zugriff auf die Zielsysteme
- Berechtigung zur Verwendung von `sudo`
- Installiertes Paketverwaltungssystem `apt`
- Definierte Variable `report_dir`
- Vorhandene Datei `mail-reporting.yml`
- Konfigurierte Möglichkeit zum E-Mail-Versand

Beispiel für die Definition von `report_dir`:

```yaml
report_dir: "/opt/ansible/reports"
```

---

## Grundkonfiguration

```yaml
- name: update
  hosts: all
  become: true
  gather_facts: true
```

| Einstellung | Beschreibung |
|---|---|
| `hosts: all` | Das Playbook wird auf allen Hosts des Ansible-Inventars ausgeführt. |
| `become: true` | Administrative Aufgaben werden mit erhöhten Rechten über `sudo` ausgeführt. |
| `gather_facts: true` | Ansible sammelt automatisch Systeminformationen der Zielsysteme. |

---

## Ablauf

Der Ablauf des Playbooks ist:

1. Ubuntu-Version vor dem Update ermitteln
2. Kernel-Version vor dem Update ermitteln
3. Paketquellen aktualisieren
4. Anzahl der verfügbaren Updates ermitteln
5. Verfügbare Updates auflisten
6. Normale Paketupdates installieren
7. Verfügbares Ubuntu-Release-Upgrade prüfen
8. Ergebnis der Release-Upgrade-Prüfung auswerten
9. Ubuntu-Version nach dem Update ermitteln
10. Kernel-Version nach dem Update ermitteln
11. Erforderlichen Neustart prüfen
12. Report-Zeitstempel einmalig festlegen
13. Report-Dateipfad setzen
14. Report-Verzeichnis erstellen
15. Update-Report erzeugen
16. Report per E-Mail versenden

---

## Eingabevariablen

| Variable | Typ | Beschreibung | Standardwert |
|---|---|---|---|
| `report_dir` | String | Verzeichnis, in dem der Report gespeichert wird | kein Standardwert |
| `perform_release_upgrade` | Boolean | Aktiviert oder deaktiviert ein Ubuntu-Release-Upgrade | `false` |
| `mail_report_file` | String | Pfad zur Report-Datei | wird automatisch gesetzt |
| `mail_subject` | String | Betreff der E-Mail | `Ubuntu Update Report` |
| `mail_to` | Liste | Empfänger der E-Mail | im Playbook definiert |
| `mail_body` | String | Nachrichtentext der E-Mail | im Playbook definiert |
| `mail_delete_report_after_send` | Boolean | Löscht den Report nach dem Versand | `true` |

### Beispiel

```yaml
report_dir: "/opt/ansible/reports"
perform_release_upgrade: false
```

> Hinweis: `perform_release_upgrade: true` reicht allein noch nicht aus. Der Task für das eigentliche Release-Upgrade muss zusätzlich im Playbook einkommentiert werden.

---

## Beschreibung der Tasks

### Ubuntu-Version vor dem Update ermitteln

```yaml
lsb_release -ds
```

Ermittelt die installierte Ubuntu-Version vor dem Update.

Beispiel:

```text
Ubuntu 22.04.5 LTS
```

Das Ergebnis wird in der Variable `ubuntu_version_before` gespeichert.

---

### Kernel-Version vor dem Update ermitteln

```yaml
uname -r
```

Ermittelt die aktuell laufende Kernel-Version vor dem Update.

Beispiel:

```text
5.15.0-161-generic
```

Das Ergebnis wird in `kernel_before` gespeichert.

---

### Paketquellen aktualisieren

```yaml
ansible.builtin.apt:
  update_cache: true
```

Aktualisiert die lokalen Paketinformationen. Dies entspricht ungefähr:

```bash
sudo apt update
```

Dabei werden noch keine Pakete installiert.

Das Ergebnis wird in `apt_update_result` gespeichert.

---

### Anzahl verfügbarer Updates ermitteln

```bash
apt list --upgradable
```

Ermittelt, wie viele Pakete aktualisiert werden können.

Das Ergebnis wird in `updates_available_before` gespeichert.

Beispiel:

```text
12
```

---

### Verfügbare Updates auflisten

Ermittelt die konkreten Pakete, für die Updates verfügbar sind.

Das Ergebnis wird in `updates_list_before` gespeichert.

Beispiel:

```text
curl/jammy-updates
openssl/jammy-updates
linux-generic/jammy-updates
```

Die Liste wird derzeit zwar gespeichert, aber nicht vollständig im Report ausgegeben.

---

### Normale Paketupdates installieren

```yaml
ansible.builtin.apt:
  upgrade: yes
```

Installiert die verfügbaren normalen Paketupdates. Dies entspricht ungefähr:

```bash
sudo apt upgrade -y
```

Das Ergebnis wird in `apt_upgrade_result` gespeichert.

Ein vollständiges Distribution-Upgrade wird durch diesen Task nicht durchgeführt.

---

### Ubuntu-Release-Upgrade prüfen

```bash
do-release-upgrade -c
```

Prüft, ob eine neue Ubuntu-Hauptversion verfügbar ist.

Beispiel:

```text
New release '24.04.3 LTS' available.
```

Die Prüfung führt noch kein Release-Upgrade durch.

Das Ergebnis wird in `release_upgrade_check` gespeichert.

---

### Verfügbares Release-Upgrade auswerten

Der Inhalt von `release_upgrade_check` wird nach einer Zeile mit `New release` durchsucht.

Wird eine neue Ubuntu-Version gefunden, wird diese in `available_release_upgrade` gespeichert.

Wird keine neue Version gefunden, wird ein entsprechender Status gesetzt.

---

### Release-Upgrade ausführen

Der Task zum eigentlichen Release-Upgrade ist derzeit auskommentiert:

```yaml
do-release-upgrade -f DistUpgradeViewNonInteractive
```

Würde der Task aktiviert und die Variable `perform_release_upgrade` auf `true` gesetzt, könnte ein Ubuntu-Release-Upgrade automatisch ausgeführt werden.

Ein Release-Upgrade sollte vorher getestet werden. Drittanbieter-Pakete, individuelle Konfigurationen oder offene Dialogfragen können den automatischen Ablauf beeinflussen.

---

### Release-Upgrade-Status bestimmen

Der Status des Release-Upgrades wird in `release_upgrade_status` gespeichert.

Mögliche Zustände sind:

| Status | Bedeutung |
|---|---|
| `deactivated` | Release-Upgrade ist deaktiviert |
| `no upgrade` | Kein Release-Upgrade verfügbar |
| `executed` | Release-Upgrade wurde erfolgreich ausgeführt |
| `error` | Fehler beim Release-Upgrade |

---

### Ubuntu-Version nach dem Update ermitteln

Ermittelt erneut die Ubuntu-Version mit:

```bash
lsb_release -ds
```

Das Ergebnis wird in `ubuntu_version_after` gespeichert.

Bei einem normalen Paketupdate bleibt die Ubuntu-Version normalerweise unverändert. Bei einem erfolgreichen Release-Upgrade ändert sie sich.

---

### Kernel-Version nach dem Update ermitteln

Ermittelt die aktuell laufende Kernel-Version mit:

```bash
uname -r
```

Das Ergebnis wird in `kernel_after` gespeichert.

> Wird durch das Paketupdate ein neuer Kernel installiert, ist dieser möglicherweise erst nach einem Neustart aktiv.

---

### Neustartbedarf prüfen

Das Playbook prüft, ob folgende Datei vorhanden ist:

```text
/var/run/reboot-required
```

Ist die Datei vorhanden, wird davon ausgegangen, dass ein Neustart erforderlich ist.

Das Ergebnis wird in `reboot_required` gespeichert.

Die eigentliche Auswertung erfolgt über:

```yaml
reboot_required.stat.exists
```

---

### Server neu starten

Der Neustart-Task ist derzeit auskommentiert:

```yaml
ansible.builtin.reboot:
  reboot_timeout: 1800
```

Würde der Task aktiviert, würde der Server nur dann neu gestartet werden, wenn `/var/run/reboot-required` vorhanden ist.

Der Neustart kann bis zu 1.800 Sekunden dauern.

---

## Report-Erstellung

Der Report wird zentral auf dem Ansible-Control-Host erstellt:

```yaml
delegate_to: localhost
become: false
run_once: true
```

Das bedeutet:

- Der Report wird nicht auf jedem Zielserver einzeln erstellt.
- Die Erstellung erfolgt auf dem Control-Host.
- Die Ergebnisse aller Zielserver werden in einer gemeinsamen Datei zusammengefasst.
- Der Report wird nur einmal erstellt.

---

### Report-Zeitstempel festlegen

Der Zeitstempel wird einmalig in `report_timestamp` gespeichert.

Beispiel:

```text
2026-09-29_14-35-22
```

Der Zeitstempel wird später für den Dateinamen verwendet.

---

### Report-Dateipfad setzen

Der vollständige Pfad der Report-Datei wird in `report_file` gespeichert.

Beispiel:

```text
/opt/ansible/reports/ubuntu-update-report-2026-09-29_14-35-22.txt
```

Der Dateiname besteht aus:

```text
ubuntu-update-report-<Zeitstempel>.txt
```

---

### Report-Verzeichnis erstellen

Das Verzeichnis aus `report_dir` wird automatisch erstellt, falls es noch nicht existiert.

Verwendete Berechtigungen:

```text
0755
```

Beispiel:

```text
/opt/ansible/reports
```

---

### Update-Report erzeugen

Der Report wird als Textdatei erstellt und enthält Informationen zu allen Zielservern.

Pro Host werden folgende Werte ausgegeben:

- Hostname
- Ubuntu-Version vor dem Update
- Ubuntu-Version nach dem Update
- Kernel-Version nach dem Update
- verfügbares Release-Upgrade
- Status des Release-Upgrades
- Anzahl der verfügbaren Updates
- Neustartstatus
- Autoremove-Status

Zusätzlich enthält der Report Zusammenfassungen über:

- Server mit verfügbaren Paketupdates
- Server mit verfügbarem Release-Upgrade
- Server mit durchgeführtem Release-Upgrade
- Server mit Release-Upgrade-Fehlern
- Server mit erforderlichem oder durchgeführtem Neustart
- Server mit Autoremove-Änderungen

---

## Verwendete Ergebnisvariablen

| Variable | Beschreibung |
|---|---|
| `ubuntu_version_before` | Ergebnis der Ubuntu-Versionsabfrage vor dem Update |
| `ubuntu_version_after` | Ergebnis der Ubuntu-Versionsabfrage nach dem Update |
| `kernel_before` | Ergebnis der Kernel-Abfrage vor dem Update |
| `kernel_after` | Ergebnis der Kernel-Abfrage nach dem Update |
| `apt_update_result` | Ergebnis der Aktualisierung der Paketquellen |
| `apt_upgrade_result` | Ergebnis des normalen Paketupdates |
| `updates_available_before` | Anzahl der verfügbaren Paketupdates |
| `updates_list_before` | Liste der verfügbaren Paketupdates |
| `release_upgrade_check` | Ausgabe der Release-Upgrade-Prüfung |
| `available_release_upgrade` | Verfügbare Ubuntu-Release-Version |
| `release_upgrade_status` | Ausgewerteter Release-Upgrade-Status |
| `reboot_required` | Ergebnis der Neustartprüfung |
| `report_timestamp` | Zeitstempel für den Report |
| `report_file` | Vollständiger Report-Dateipfad |

---

## E-Mail-Versand

Der E-Mail-Versand erfolgt über die eingebundene Datei:

```yaml
mail-reporting.yml
```

Die Datei wird mit folgendem Task eingebunden:

```yaml
ansible.builtin.include_tasks: mail-reporting.yml
```

Dabei werden unter anderem folgende Variablen übergeben:

```yaml
mail_report_file: "{{ report_file }}"
mail_subject: "Ubuntu Update Report"
mail_to:
  - "jannik.janzen@schuetz.net"
mail_body: |
  Aktueller Ubuntu Update Report
mail_delete_report_after_send: true
```

Der Versand wird nur einmal durchgeführt:

```yaml
run_once: true
```

Nach dem Versand soll die Report-Datei gelöscht werden. Ob dies tatsächlich geschieht, hängt von der Implementierung in `mail-reporting.yml` ab.

---

## Derzeit deaktivierte Funktionen

Die folgenden Funktionen sind im Playbook auskommentiert:

- Vollständiges Distribution-Upgrade mit `upgrade: dist`
- Entfernen nicht benötigter Pakete mit `autoremove`
- Automatisches Ubuntu-Release-Upgrade
- Automatischer Neustart

Aktuell wird daher nur ein normales Paketupdate durchgeführt. Release-Upgrade und Neustart werden lediglich geprüft.

---

## Bekannte Punkte

### Unterschiedliche Statuswerte

Im Playbook und im Report werden teilweise unterschiedliche Statuswerte verwendet.

| Im Playbook | Im Report erwartet |
|---|---|
| `executed` | `Durchgeführt` |
| `error` | `Fehler` |
| `No Upgrade avialable` | `Kein Upgrade verfügbar` |

Dadurch können bestimmte Bereiche im Report leer bleiben, obwohl ein Status vorhanden ist.

Die Werte sollten vereinheitlicht werden.

### Schreibfehler

Der Wert:

```text
No Upgrade avialable
```

enthält einen Schreibfehler.

Korrekt wäre:

```text
No Upgrade available
```

Da der Report deutschsprachig ist, empfiehlt sich ein einheitlicher deutscher Wert:

```text
Kein Upgrade verfügbar
```

### Variable `report_dir`

Die Variable `report_dir` muss vor dem Playbook-Lauf definiert werden. Andernfalls kann der Report-Dateipfad nicht erstellt werden.

### Liste der verfügbaren Updates

Die Variable `updates_list_before` wird ermittelt, aber aktuell nicht vollständig im Report angezeigt.

### Kernel nach dem Update

Wird ein neuer Kernel installiert, ist dieser möglicherweise erst nach einem Neustart aktiv. Die angezeigte Kernel-Version kann deshalb vor dem Neustart noch die alte Version sein.

---

## Beispiel für `group_vars/all.yml`

```yaml
report_dir: "/opt/ansible/reports"
perform_release_upgrade: false
```

---

## Gesamtablauf als Übersicht

```text
Ubuntu- und Kernel-Version vor dem Update erfassen
                    ↓
Paketquellen aktualisieren
                    ↓
Verfügbare Updates zählen und auflisten
                    ↓
Normale Paketupdates installieren
                    ↓
Release-Upgrade prüfen
                    ↓
Ubuntu- und Kernel-Version nach dem Update erfassen
                    ↓
Neustartbedarf prüfen
                    ↓
Report-Zeitstempel festlegen
                    ↓
Report-Dateipfad setzen
                    ↓
Report-Verzeichnis erstellen
                    ↓
Update-Report erzeugen
                    ↓
Report per E-Mail versenden
                    ↓
Report gegebenenfalls löschen
```

---