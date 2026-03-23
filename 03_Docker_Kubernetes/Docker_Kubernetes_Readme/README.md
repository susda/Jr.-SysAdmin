# Homelab Phase 3: Containerisierung & Orchestrierung

Dieses Repository dokumentiert die dritte Phase meines Homelab-Projekts: den Aufbau einer stabilen, sicheren und skalierbaren containerbasierten Infrastruktur auf Basis von **Docker**, **Docker Compose** und **Portainer**. Der Fokus liegt auf den Prinzipien von *Infrastructure as Code* (IaC), Netzwerkisolierung und *Disaster Recovery Protocols*.

### Projektziele

* ✅ **Imperativ zu Deklarativ:** Migration von manuellen `docker run` Befehlen zu deklarativen YAML-Definitionen (Infrastructure as Code).
* ✅ **Netzwerkisolierung (Security):** Implementierung isolierter Backend-Netzwerke zum Schutz sensibler Microservices.
* ✅ **Datenpersistenz:** Sicherstellung, dass kritische Daten (Datenbanken, Webdateien) Container-Neustarts und Host-Löschungen überleben.
* ✅ **Disaster Recovery Protokoll:** Etablierung verifizierter Backup- und Wiederherstellungsverfahren.
* ✅ **Multi-Node Orchestrierung:** Aufbau eines Mini-Clusters unter Verwendung der Portainer Agent Architektur.

---

## 🏛️ Schematische Übersicht der Kernkonzepte

Um die technischen Zusammenhänge besser zu verstehen, haben wir die drei wichtigsten Lernschritte visuell zusammengefasst:



---

## Meilenstein 1: Single-Node Docker & Disaster Recovery

Im ersten Schritt haben wir die Laufzeitumgebung stabilisiert und eine komplexe Webanwendung („Das Bankhaus“) auf einem einzigen Knoten aufgebaut.

### 1.1 Container Runtime & Identity Management
Bevor wir Dienste bereitstellen konnten, mussten wir die logische Identität des Pi 5 Knoten konfigurieren und die Docker Engine installieren.

* **Identität:** Wir haben flüchtige IPs eliminiert und eine logische Nomenklatur (`pi5-node-01`) via `/etc/hosts` und `hostnamectl` etabliert.
* **Installation:** Bereitstellung des offiziellen Docker GPG Keys in `/etc/apt/keyrings/docker.gpg` und Installation der Enterprise Artefakte (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-compose-plugin`).

### 1.2 "Das Bankhaus": Netzwerkisolierung & Sicherheit
Wir deployten eine Multi-Container-Anwendung (WordPress + MariaDB) unter Verwendung des Konzepts „Das Bankhaus“, um den Datenfluss und die Sicherheitsschichten zu demonstrieren.

* **Der Kassierer (Frontend - WordPress):** Hört intern auf Port 80 und ist über Port 8082 von außen (MacBook Air) erreichbar.
* **Der Tresor (Backend - MariaDB):** Kommuniziert **ausschließlich** intern im isolierten `bank_net` Netzwerk auf Port 3306. **Keine** Ports nach außen exponiert. Angriffsfläche drastisch reduziert.

> *Siehe visuelle Erklärung in Panel 1: "NETZWERKISOLIERUNG ('DAS BANKHAUS')".*

**Validierung des Deployments:**
<img width="1024" height="559" alt="Validation des hello-world Containers" src="./images/image_e4199f.png" />

### 1.3 Datenpersistenz & Berechtigungen
Standard-Docker-Volumes (`db_data`, `wp_data`) wurden konfiguriert, damit Daten Neustarts überleben. Wir mussten die Berechtigungen (`chown`) der Linux-Ordner manuell anpassen, damit Docker korrekt in die Volumes schreiben konnte.

### 1.4 Disaster Recovery Protokoll (Lessons Learned)
Dies war der kritischste Lernmoment. Wir simulierten einen totalen Datenverlust (`docker compose down -v`) und eine fehlerhafte Wiederherstellung, bevor wir das verifizierte Protokoll etablierten.

> *Siehe visuelle Erklärung in Panel 2: "DISASTER RECOVERY PROTOKOLL".*

#### Die Herausforderung: Das fehlerhafte Backup
Wir vertrauten auf ein Backup, das mit dem falschen Befehl (`mariadb_dump` statt `mysqldump`) erstellt wurde. Es enthielt nur Docker-Fehlermeldungen. Als wir die Volumes löschten, waren alle Daten verloren.

#### Die Lösung: Das verifizierte Protokoll
1.  **Backup:** Nutzung des verifizierten `mysqldump` Befehls.
    ```bash
    docker exec vault_db mysqldump -u wp_user -pwp_password_secreto wordpress_db > backup.sql
    ```
2.  **Verifizierung:** Prüfung des Inhalts mit `head` (Echter SQL Code vs. Fehlertext).
3.  **Deterministische Wiederherstellung:**
    ```bash
    cat backup.sql | docker exec -i vault_db mariadb -u wp_user -pwp_password_secreto wordpress_db
    ```

---

## Meilenstein 2: Multi-Node Orchestrierung & Stacks

Jetzt, da wir wissen, wie man einen einzelnen Knoten sichert und wiederherstellt, ist das Ziel, die Infrastruktur **ausfallsicher** zu machen und die **Raspberry Pi 4** als Worker-Knoten zu integrieren.

### 2.1 Aufbau eines Mini-Clusters mit Portainer

Wir verwenden den Raspberry Pi 5 als "Kontrollturm" (Manager Node) und den Raspberry Pi 4 als "Arbeitsknoten" (Worker Node). Die Kommunikation erfolgt über die Portainer Agent Architektur.

> *Siehe visuelle Erklärung in Panel 3: "MULTI-NODE CLUSTER (PORTAINER AGENT ARCHITEKTUR)".*

**Schritt 1: Den Worker-Knoten vorbereiten (Raspberry Pi 4)**
Wir installieren den Portainer Agent: 
```bash
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest


version: '3.8' # Compose V3 API Standard

services:
  web-diagnostico:
    image: nginxdemos/hello:latest # Leichtgewichtiger Web-Server
    container_name: app-diagnostico-pi4
    restart: unless-stopped # Hohe Ausfallsicherheit: Neustart nach Reboot des Hosts
    ports:
      - "8085:80" # Host-Port 8085 -> Container-Port 80


Konzept,Imperativ (docker run),Deklarativ (docker compose)
Befehl,docker run -d --name mi-server -p 8080:80 ...,docker compose up -d
Dokumentation,"""Verloren"" im Terminalverlauf (unübersichtlich)",Gespeichert in der docker-compose.yml (IaC)
Änderungen,Neustart mit neuem Befehl (komplex),"nano Änderung, dann docker compose up -d"
Status,Unübersichtlich,docker compose ps (Health Checks)

# 1. SSH-Verbindung zum Master-Node (Public-Key-Authentifizierung)
ssh robert@pi5-node-01.local

# 2. Erstellung der Verzeichnisstruktur (-p für Parents)
mkdir -p ~/opt/webserver/html

# 3. Erstellung der IaC-Datei im Terminal
nano docker-compose.yml

# 4. Ausführung des deklarativen Deployments im Detached Mode (-d)
docker compose up -d

# 5. Echtzeit-Einsicht in Container-Logs zur Fehlerbehebung
docker logs -f vault_db

# 6. SQL-Validierungsbefehle (Interaktive Exec)
docker exec -it vault_db mariadb -u wp_user -p
show databases;


# 🚀 Phase 4: K3s Multi-Node Cluster (High Availability)

Nach der erfolgreichen Implementierung von Docker auf einem einzelnen Knoten haben wir heute die Infrastruktur auf ein **verteiltes System** skaliert. Wir nutzen **K3s** (Lightweight Kubernetes), um die Rechenleistung der Raspberry Pi 5 und Raspberry Pi 4 zu einem Cluster zu bündeln.

---

## 🏛️ Cluster-Architektur: Master & Worker

In dieser Konfiguration übernimmt die **Raspberry Pi 5** die Rolle des "Gehirns" (Control Plane), während die **Raspberry Pi 4** als "Muskel" (Worker Node) fungiert.



* **Control Plane (Master - Pi 5):** Verantwortlich für das API-Management, Scheduling und die Verwaltung des Cluster-Zustands.
* **Worker Node (Agent - Pi 4):** Führt die eigentlichen Anwendungs-Container (Pods) aus.
* **Kubelet:** Der Agent auf dem Worker-Node, der die Befehle des Masters entgegennimmt.

---

## 🛠️ Schritt-für-Schritt Implementierung

### 1. Kernel-Optimierung (Cgroups)
Kubernetes benötigt eine strikte Ressourcenkontrolle. Da dies bei Raspberry Pis standardmäßig deaktiviert ist, mussten wir die Kernel-Parameter anpassen.

* **Datei:** `/boot/firmware/cmdline.txt`
* **Parameter:** `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`



### 2. Initialisierung des Masters (Pi 5)
Die Installation erfolgte über das offizielle K3s-Skript. Nach dem Setup haben wir den Sicherheits-Token extrahiert, um den Worker-Node zu autorisieren.

```bash
# Installation auf dem Master
curl -sfL [https://get.k3s.io](https://get.k3s.io) | sh -

# Abrufen des Join-Tokens
sudo cat /var/lib/rancher/k3s/server/node-token

3. Integration des Workers (Pi 4)
Der Pi 4 wurde mittels Token und Master-IP in den Cluster integriert. Ein kritischer Punkt war hier die vorherige Bereinigung einer fehlerhaften Standalone-Installation mittels k3s-uninstall.sh.

# Beitritt zum Cluster vom Pi 4 aus
curl -sfL [https://get.k3s.io](https://get.k3s.io) | K3S_URL=[https://192.168.0.20:6443](https://192.168.0.20:6443) K3S_TOKEN=<TOKEN> sh -

🔍 Troubleshooting: "Exec Format Error" & Multi-Arch
Beim ersten Deployment-Test stießen wir auf einen klassischen Architektur-Fehler: exec format error.

Problem: Das gewählte Image (hello-kubernetes:1.10) war nicht für die 64-Bit ARM-Architektur der Pi 5 optimiert.

Lösung: Umstellung auf Multi-Arch-Images (nginx:alpine). Diese enthalten Manifeste für verschiedene CPU-Typen (amd64, arm64, arm/v7) und garantieren die Lauffähigkeit auf beiden Knoten.

Validierung des Workloads
Wir haben ein Deployment mit 3 Replikaten erstellt, um die Verteilung (Scheduling) zu prüfen:

# Befehl zur Prüfung der Verteilung
sudo kubectl get pods -o wide
![alt text](image-1.png)

📈 Fazit & Lernerfolg
Durch den Aufbau dieses Clusters haben wir die Abhängigkeit von einer einzelnen Maschine eliminiert. Wir haben gelernt:

Wie Kubernetes Entscheidungen über das Scheduling trifft.

Warum CPU-Architekturen (arm64 vs. arm/v7) bei der Image-Wahl entscheidend sind.

Wie man ein Self-Healing-System konfiguriert, das Workloads automatisch neu verteilt.
