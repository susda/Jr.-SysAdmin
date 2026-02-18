
# SysAdmin & Cloud Ops Journey: Vom Homelab zur Produktion

Willkommen in meinem Dokumentations-Repository.
Dieses Projekt dokumentiert meinen intensiven 8-Wochen-Pfad zum **Junior Systemadministrator / Cloud Operator** in der Rhein-Main-Region.

Das Ziel ist nicht nur das "Ausprobieren" von Software, sondern der Aufbau einer **professionellen, hybriden Infrastruktur**, die reale Unternehmensanforderungen simuliert (Sicherheit, Monitoring, Backups).

---

## 📅 Projekt-Logbuch: Tag 1
**Thema:** Fundament & Virtualisierung
**Status:** ✅ Abgeschlossen

### 🎯 Ziele des Tages
1.  **Netzwerk-Segmentierung (Layer 2):** Einrichtung eines Managed Switch zur Trennung von Labor- und Heimnetzwerk.
2.  **Bare-Metal Virtualisierung:** Installation von **Proxmox VE** auf dedizierter Intel-Hardware (kein Dual-Boot, volle Performance).
3.  **Edge-Computing Vorbereitung:** Setup der Raspberry Pi 5 (NVMe Boot) und Pi 4 als zukünftiges Kubernetes-Cluster.

### 🛠️ Technische Umsetzung

#### 1. Der Hypervisor (Proxmox VE)
Anstatt VirtualBox auf einem Desktop-OS zu nutzen, habe ich einen **Typ-1 Hypervisor** aufgesetzt.
* **Hardware:** Intel i9 (13th Gen), 32GB RAM.
* **Storage-Strategie:** Trennung von OS (250GB NVMe) und VM-Storage (1TB NVMe) für maximale I/O-Performance.
* **Netzwerk:** Statische IP (`.10`) und DNS-Konfiguration via Cloudflare.

#### 2. Das ARM-Cluster (Raspberry Pi)
Vorbereitung für Container-Orchestrierung.
* **Pi 5 (Master):** Konfiguration des Bootloaders für **NVMe-Boot** (PCIe Gen 2/3).
* **Pi 4 (Worker):** Setup mit High-End SD-Speicher (A2 Standard).
* **Sicherheit:** Deaktivierung der Passwort-Authentifizierung. Zugang ausschließlich via **SSH-Keys (Ed25519)**.

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/73367410-10c1-412d-a93f-afb9a43e219b" />


## 🛠️ Phase 1: OS-Installation, SSH-Hardening & Netzwerk-Diagnose

Der erste Schritt bestand darin, die Nodes (Raspberry Pi 4 & 5) mit einem minimalen Betriebssystem zu provisionieren und den sicheren Fernzugriff zu gewährleisten. Dabei kamen fortgeschrittene Diagnose-Tools zum Einsatz, um Verbindungskonflikte während der Hardware-Wechsel zu lösen.

### 1. Image-Provisionierung (Raspberry Pi Imager)
Für die Erstinstallation wurde **Raspberry Pi OS Lite (64-bit)** gewählt, um Ressourcen zu sparen (Headless-Betrieb). Über die "OS Customization"-Funktion des Imagers wurden vorab folgende Parameter injiziert:
* **Hostname:** `pi5-node-01` / `pi4-node-01`
* **SSH:** Aktiviert (Public-Key Authentication only).
* **User:** `robert`
* **Locale:** Europe/Berlin.

### 2. Advanced Connectivity Troubleshooting (Deep Dive)
Während des Wechsels zwischen Boot-Medien (NVMe vs. SD-Karte) traten Konflikte bei der Host-Identifizierung auf. Da sich die "Hardware-Identität" (Host Key) änderte, verweigerte der Client (macOS) die Verbindung. Zur Lösung wurden folgende Methoden angewandt:

#### 🕵️‍♂️ Verbose-Debugging (`ssh -v`)
Anstatt nur auf "Connection Refused" zu starren, wurde der SSH-Client im Verbose-Mode ausgeführt:
''bash
ssh -v robert@192.168.0.20''


## 📡 Phase 2: Netzwerk-Provisionierung, Storage-Setup & Disaster Recovery

Nach der physischen Montage bestand das Ziel darin, eine statische Netzwerkkonfiguration zu etablieren und den NVMe-Boot für den Master-Knoten einzurichten. Während dieses Prozesses traten kritische Authentifizierungs- und Boot-Konflikte auf, die Eingriffe auf Low-Level-Ebene erforderten.

### 1. Definition der Netzwerkarchitektur (Static Networking)
**Ziel:** Migration von dynamischer DHCP-Zuweisung zu statischer IP-Adressierung, um die Persistenz und Erreichbarkeit der Cluster-Dienste zu gewährleisten.

* **Gateway:** `192.168.0.1`
* **Proxmox (Hypervisor):** `192.168.0.10`
* **Raspberry Pi 5 (Master):** `192.168.0.20`
* **Raspberry Pi 4 (Worker):** `192.168.0.30`

**Implementierung:** Manuelle Konfiguration via `nmtui` (NetworkManager) auf jedem Knoten sowie Deaktivierung des DHCP-Managements für die eth0-Schnittstellen zur Vermeidung von IP-Drift.

Layer-2 Cache Flushing (arp)
Durch den Wechsel der Boot-Medien (und teilweise der MAC-Adressen-Zuordnung) enthielt der ARP-Cache des Clients veraltete Einträge.
sudo arp -d -a

🔑 Host Key Management (ssh-keygen)
Beim Wechsel vom NVMe-System zum Rescue-SD-System warnte SSH vor einem "Man-in-the-Middle"-Angriff (WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!), da sich der Fingerprint des Servers geändert hatte.

ssh-keygen -R 192.168.0.20

---

### 2. Post-Mortem: Kritischer Vorfall (Zugriffssperre & NVMe-Boot-Fehler)
Während der initialen Provisionierung des Master-Nodes (Pi 5) auf der NVMe-SSD ging der administrative Zugriff vollständig verloren.

#### 🔴 Das Problem
Das auf der NVMe installierte Betriebssystem lehnte den konfigurierten öffentlichen SSH-Key ab (`Permission denied (publickey)`). Da es sich um eine "Headless"-Installation (ohne Monitor) handelte und aus Sicherheitsgründen kein Passwort-Login aktiviert war, befand sich das System in einem "Locked Out"-Status.

#### 🔍 Technische Analyse (Root Cause Analysis)
* **Ursache:** Fehler bei der Injektion der `authorized_keys` oder Konfigurationsfehler während des Flash-Vorgangs des NVMe-Images.
* **Erschwerender Faktor:** Der Raspberry Pi priorisierte den Boot-Vorgang von der korrupten NVMe, was einen Zugriff zur Reparatur verhinderte.
* **Physische Limitierung:** Es war kein externer NVMe-Adapter verfügbar, um den Datenträger an einem externen Host (Mac) zu mounten und zu reparieren.

#### 🛠️ Lösungsstrategie: "Rescue System Technique"
Entwurf einer **"Physical Bypass"**-Strategie, um die Hardware mittels eines sekundären Speichermediums (MicroSD) zu booten und eine Systemreparatur am NVMe-Datenträger "in-place" durchzuführen.

#### ✅ Technische Umsetzung (Schritt-für-Schritt)
1.  **Erstellung eines Rettungsmediums:** Flashen einer SD-Karte mit Raspberry Pi OS Lite und bekannten Backup-Credentials (User/Password).
2.  **Modifikation des Bootloaders (EEPROM):** Erzwingen der Boot-Priorität via `sudo rpi-eeprom-config`. Setzen von `BOOT_ORDER=0xf1` (SD-Karte priorisieren, bei Fehler Neustart), um die fehlerhafte NVMe beim Start zu ignorieren.
3.  **Block-Level-Cloning (`dd`):** Nach erfolgreichem Boot in das Rettungssystem wurde die Integrität der NVMe via `lsblk` geprüft. Anschließend wurde das funktionierende Live-System bitgenau auf die NVMe geklont:
    ```bash
    sudo dd if=/dev/mmcblk0 of=/dev/nvme0n1 bs=4M status=progress
    ```
4.  **Wiederherstellung der Boot-Reihenfolge:** Revertierung der EEPROM-Konfiguration auf `0xf416` (Priorität SD -> NVMe) für den regulären Betrieb.
5.  **Dateisystem-Erweiterung (Filesystem Expand):** Nach dem Reboot von der geklonten NVMe wurde `raspi-config` genutzt, um die Partitionstabelle zu vergrößern und den ungenutzten Speicherplatz der 250GB SSD freizugeben.

---

### 🎓 Wichtige Erkenntnisse (Key Takeaways)
* **Bootloader-Management:** Verständnis der EEPROM-Konfiguration bei ARM-Architekturen zur Steuerung der Boot-Sequenz.
* **Linux Storage Handling:** Einsatz von Low-Level-Tools (`lsblk`, `dd`) zur Datenträgeranalyse und zum Klonen von Live-Systemen.
* **Netzwerk-Troubleshooting:** Diagnose von IP-Konflikten und Bereinigung von ARP/Known_hosts-Caches.
* **Resilienz:** Bedeutung redundanter Zugriffsmethoden (Console/Password) während der initialen Setup-Phasen.

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/8e84e304-ab54-45d9-9f2b-f1c488e0fcb7" />

## 🐳 Phase 3: Container Runtime & Identity Management

Nach der Stabilisierung der Hardware-Ebene (Layer 1) und des Betriebssystems (Layer 2) erfolgte die Konfiguration der logischen Identität und die Implementierung der Container-Virtualisierungsschicht.

### 1. DNS & Hostname Konfiguration (Local Resolution)
**Ziel:** Eliminierung der Abhängigkeit von flüchtigen IPs und Etablierung einer logischen Nomenklatur (`node-01`, `proxmox`) ohne externen DNS-Resolver.

* **Netzwerklogik:** Implementierung einer statischen Namensauflösung via `/etc/hosts` auf allen Cluster-Nodes (Client macOS, Master, Worker).
* **Identitäts-Persistenz:** Permanente Änderung der Hostnamen mittels `hostnamectl` und `sed` zur Ablösung der generischen Defaults (`raspberrypi`) durch Infrastruktur-IDs (`pi5-node-01`).
* **Ergebnis:** Reibungslose Node-to-Node Kommunikation via FQDN-Simulation.
    * `ping pi5-node-01` ✅
    * `ping pi4-node-01` ✅

### 2. Installation der Docker Engine (Enterprise Standard)
**Ziel:** Bereitstellung einer sicheren, aktuellen Container-Laufzeitumgebung (Runtime), unter Umgehung der oft veralteten Debian-Repository-Pakete.

* **Security & Trust (Keyrings):**
    * Download und Installation des offiziellen **Docker GPG Keys** in `/etc/apt/keyrings/docker.gpg`.
    * *Grund:* Gewährleistung der Paket-Authentizität und Schutz der Supply Chain.
* **Repository-Konfiguration:**
    * Dynamische Injektion des offiziellen Docker-Upstream-Repos unter Berücksichtigung der CPU-Architektur (`arm64`) und OS-Version (`bookworm`).
* **Installierte Artefakte:**
    * `docker-ce`: Die Core Engine.
    * `docker-ce-cli`: Command Line Interface.
    * `containerd.io`: Industriestandard-Runtime.
    * `docker-buildx-plugin` & `docker-compose-plugin`: Erweiterungen für Multi-Arch-Builds und Orchestrierung.

### 3. Rechteverwaltung (Privilege Management)
**Ziel:** Operatives Container-Management ohne permanente Root-Rechte (Least Privilege Principle).

* **Maßnahme:** Hinzufügen des administrativen Users `robert` zur Systemgruppe `docker`.
    * Befehl: `sudo usermod -aG docker $USER`
* **Technischer Effekt:** Gewährung von Lese-/Schreibrechten auf den Unix-Socket des Docker Daemons (`/var/run/docker.sock`).

### 4. Validierung (Smoke Test)
**Test:** Deployment des `hello-world` Containers.
**Ergebnis:** Erfolgreicher Kontakt zur Docker Registry, Layer-Pull, Container-Instanziierung und Log-Output. Bestätigung der Internet-Konnektivität und der Integrität des Daemons.

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/8522af5d-8dc5-403e-b25f-a10c0b823066" />


## 🚀 Phase 4: Web-Server Deployment & Remote-Development Workflow

Nach der Einrichtung der Docker-Engine war das Ziel, einen echten Web-Service bereitzustellen und von einer manuellen CLI-Verwaltung zu einer professionellen IDE-gestützten Entwicklungsumgebung zu wechseln.

### 1. Einrichtung der Entwicklungsumgebung (IDE Setup)
**Ziel:** Implementierung eines "Remote-SSH" Workflows, um Code lokal auf dem Mac (VS Code) zu schreiben und direkt auf der Raspberry Pi (Target) auszuführen.

* **Tooling:** Visual Studio Code + Remote-SSH Extension.
* **Konfiguration:**
    ```ssh
    # ~/.ssh/config (macOS Client)
    Host pi5-node-01
        HostName 192.168.0.20
        User robert
        IdentityFile ~/.ssh/id_rsa
    ```
* **Vorteil:** Direkte Bearbeitung von Dateien auf dem Server ohne Latenz oder VIM/Nano-Abhängigkeit.

### 2. Projektstruktur & Content
Erstellung einer statischen Web-Präsenz zur Validierung des Webservers.
* **Pfad:** `/home/robert/mainz-web/index.html`
* **Code (HTML5):**
    ```html
    <!DOCTYPE html>
    <html>
    <head>
        <title>Mainz Cluster</title>
        <style>
            body { background-color: #003366; color: #ffffff; font-family: sans-serif; text-align: center; }
        </style>
    </head>
    <body>
        <h1>Mainz Cluster v2.0</h1>
        <p>Deployed via Docker Container on Raspberry Pi 5</p>
        <p>Status: <strong>OPERATIONAL</strong></p>
    </body>
    </html>
    ```

### 3. Container Deployment (Nginx)
Bereitstellung des Webservers unter Verwendung von **Bind Mounts** und **Port Mapping**.

**Docker Befehl:**
```bash
docker run -d \
  --name mi-web-server \
  -p 8080:80 \
  -v ~/mainz-web/index.html:/usr/share/nginx/html/index.html:ro \
  nginx
```
<img width="1462" height="532" alt="image" src="https://github.com/user-attachments/assets/762c3f0c-7716-491b-9dbc-6479a81a0596" />

🚀 Praxis-Guide: Aufbau eines Mini-Clusters mit Portainer
Das Ziel ist es, die Verwaltung deiner Geräte zu zentralisieren. Wir verwenden den Raspberry Pi 5 als "Kontrollturm" (Manager) und den Raspberry Pi 4 als "Arbeitsknoten" (Worker).

📋 Voraussetzungen
Docker ist auf beiden Raspberry Pis installiert.

Portainer läuft auf dem Pi 5 (Port 9443).

SSH-Zugang zum Pi 4.

🛠️ Schritt 1: Den Worker-Knoten vorbereiten (Raspberry Pi 4)
Damit der Pi 5 dem Pi 4 Befehle erteilen kann, müssen wir einen "Agenten" installieren.

Verbinde dich mit dem Pi 4 über SSH.

Überprüfe Docker:

Bash
docker --version
Installiere den Portainer Agent:
Führe diesen Befehl aus, um das "Kommunikationstor" zu öffnen:

Bash
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest
Ermittle die IP-Adresse des Pi 4:
Diese wird im nächsten Schritt benötigt.

Bash
hostname -I
🌉 Schritt 2: Die Knoten verknüpfen (Im Browser)
Nun konfigurieren wir den "Kontrollturm" auf dem Pi 5.

Öffne Portainer in deinem Browser: https://pi5-node-01:9443.

Gehe zu Environments > + Add environment.

Wähle Docker Standalone aus und klicke auf Connect a remote node.

Wähle die Option Agent aus.

Fülle das Feld Name mit Pi4-Worker aus.

Fülle das Feld Environment address mit [IP_DEINES_PI4]:9001 aus (Beispiel: 192.168.1.55:9001).

Klicke auf Connect.

🕹️ Schritt 3: Remote-Deployment Experiment
Wir testen nun, ob alles funktioniert, indem wir einen Container auf dem Pi 4 über das Dashboard des Pi 5 bereitstellen.

Wähle auf der Home-Seite von Portainer die neue Umgebung Pi4-Worker aus.

Gehe zu Containers > + Add container.

Setze den Name auf web-remota-pi4.

Setze das Image auf nginx:latest.

Unter Manual network port publishing: Host 8080 -> Container 80.

Klicke auf Deploy the container.

Der Härtetest: Öffne in deinem Browser http://[IP_DEINES_PI4]:8080. Wenn du "Welcome to nginx!" siehst, lebt dein Cluster!

🏛️ Architektur-Zusammenfassung
Manager-Knoten (Pi 5): Das ist das Gehirn. Hier befindet sich die Weboberfläche von Portainer, und von hier aus triffst du alle Entscheidungen.

Worker-Knoten (Pi 4): Das sind die Muskeln. Er führt die Container aus, die ihm der Manager über den Portainer Agent zuweist.

Der Vorteil: Du musst dich nicht mehr über das Terminal in jede Maschine einloggen; du verwaltest deine gesamte Infrastruktur bequem über einen einzigen Browser-Tab.

🧹 Aufräumen und nächste Schritte
Um dein System sauber zu halten, lösche den Test-Container:

Wähle in Portainer (innerhalb des Pi 4) den Container web-remota-pi4 aus.

Klicke auf Remove und bestätige.

🗺️ Was kannst du als Nächstes lernen?
Option A: Stacks (Docker Compose): Komplexe Anwendungen (mehrere Container gleichzeitig) mit Konfigurationsdateien direkt in Portainer bereitstellen.

Option B: Nginx Proxy Manager: Vergiss IP-Adressen und Ports und nutze stattdessen lokale Domainnamen wie http://pi4.local.

Verwandte Notiz: [[7 Raspy 4 -> Panel de control de Portainer]]

📚 Zum Lernen
🖥️ 1. Terminal-Theorie (Was heute hinter den Kulissen passierte)
SSH (Secure Shell): Als du dein VS Code mit dem Pi 4 (robert@pi4-node-01) verbunden hast, hast du SSH verwendet. Es ist ein Protokoll, das einen verschlüsselten Tunnel erstellt. Alles, was du tippst, reist sicher, sodass niemand in deinem WLAN-Netzwerk deine Befehle abfangen kann.

Dämonen (Daemons): Als du den Portainer-Agenten ausgeführt hast, hast du das Flag -d (Detached) hinzugefügt. In der Linux-Welt nennt man Programme, die im Hintergrund laufen, ohne dein Terminal zu blockieren, "Dämonen". Die Docker-Engine selbst ist ein Dämon (dockerd).

Der Socket (/var/run/docker.sock): Heute hast du diese Datei an den Agenten übergeben. Ein Socket in Linux ist keine normale Textdatei, sondern ein direktes "Kommunikationsrohr" zum Gehirn eines Programms. Durch das Mapping hast du dem Agenten die Hauptschlüssel für das Docker deines Pi 4 gegeben.

🌐 2. Netzwerk-Theorie (Die Brücke zwischen deinen Raspberrys)
Lokale IP-Adressen (LAN): Der Befehl hostname -I hat dir die interne IP deines Pi 4 verraten (z. B. 192.168.x.x). Diese IP ist privat und wird von deinem Heimrouter zugewiesen; sie ist aus dem Internet nicht direkt sichtbar.

Ports (Die "Wohnungen" des Servers): Dein Pi 4 hat nur eine IP (stell dir vor, das ist die Adresse des Gebäudes), aber Tausende von Ports (die Wohnungsnummern). Damit es keine Konflikte gibt, nutzt jeder Dienst einen anderen.

Port 22: Genutzt von deinem VS Code (SSH).

Port 9001: Hört auf die Befehle von Portainer (Der Agent).

Port 8080: Dorthin hast du deinen Nginx-Webserver gemappt.

TCP (Transmission Control Protocol): Das Protokoll, das du heute ausgewählt hast. TCP führt einen "3-Wege-Handschlag" (3-Way Handshake) durch, bevor Daten gesendet werden: "Bist du bereit zu empfangen? Ja, ich bin bereit. Okay, hier kommt das Paket". Es ist etwas langsamer als UDP, garantiert aber, dass die Webseite vollständig geladen wird, ohne dass auch nur ein einziger Buchstabe des Codes verloren geht


