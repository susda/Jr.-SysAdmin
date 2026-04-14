# 🚀 Homelab Projekt: Von Docker zu Kubernetes (K3s)

![Status: Production Ready](https://img.shields.io/badge/Status-Production%_Ready-success?style=for-the-badge)
![Arch: ARM64](https://img.shields.io/badge/Architektur-ARM64_(Raspberry_Pi)-blue?style=for-the-badge)
![Tech: K3s](https://img.shields.io/badge/Kubernetes-K3s-orange?style=for-the-badge)

Dieses Repository dokumentiert den Aufbau einer stabilen, sicheren und skalierbaren containerbasierten Infrastruktur in meinem Heimlabor. Das Projekt begann mit **Docker & Portainer** (Phase 3) auf einem Single-Node und entwickelte sich zu einem voll funktionsfähigen **High-Availability K3s Kubernetes Cluster** (Phase 4 & 5) bestehend aus einem Raspberry Pi 5 (Master) und einem Raspberry Pi 4 (Worker).

Der Fokus liegt auf den Prinzipien von *Infrastructure as Code* (IaC), Netzwerkisolierung, *Disaster Recovery* und moderner Microservice-Orchestrierung.

---

## 🎯 Projektziele & Meilensteine

* ✅ **Imperativ zu Deklarativ:** Migration von manuellen CLI-Befehlen zu deklarativen YAML-Definitionen (IaC).
* ✅ **Netzwerkisolierung (Security):** Isolierte Backend-Netzwerke zum Schutz sensibler Datenbank-Services.
* ✅ **Datenpersistenz:** Sicherstellung, dass kritische Daten Neustarts und Pod-Zerstörungen überleben.
* ✅ **Disaster Recovery Protokoll:** Etablierung verifizierter Backup- und Wiederherstellungsverfahren.
* ✅ **Multi-Node Orchestrierung:** Verteilung von Workloads über mehrere physische Maschinen (K3s).
* ✅ **Secret Management:** Verschlüsselung von Zugangsdaten nach Industriestandard (Base64 in K8s).
* ✅ **Load Balancing & Ingress:** Bereitstellung der Services über professionelle Domains (z.B. `elbanco.local`).

---

## 🐳 Phase 3: Single-Node Docker & Disaster Recovery

Im ersten Schritt haben wir die Laufzeitumgebung stabilisiert und eine komplexe Webanwendung („Das Bankhaus“ – WordPress + MariaDB) auf einem einzigen Knoten aufgebaut.

### 1. Container Runtime & Identity Management
Bevor wir Dienste bereitstellen konnten, mussten wir die logische Identität des Pi 5 Knoten konfigurieren:
* **Identität:** Etablierung einer logischen Nomenklatur (`pi5-node-01`) via `/etc/hosts` und `hostnamectl`, um flüchtige IPs zu eliminieren.
* **Installation:** Bereitstellung des offiziellen Docker GPG Keys und Installation der Docker Engine (`docker-ce`).

### 2. "Das Bankhaus": Netzwerkisolierung & Sicherheit
Wir deployten die Anwendung, um strikte Netzwerk- und Sicherheitsschichten zu demonstrieren.

* **Der Kassierer (Frontend - WordPress):** Exponiert in das lokale Heimnetzwerk (Port 8082).
* **Der Tresor (Backend - MariaDB):** Kommuniziert **ausschließlich** intern im isolierten `bank_net` Netzwerk. 

> 🛡️ **Security Check:** Die Datenbank hat keine offenen Ports nach außen. Die Angriffsfläche wurde drastisch reduziert.

![Screenshot: Portainer Netzwerk-Ansicht, die zeigt, wie MariaDB vom Host-Netzwerk isoliert ist](./images/docker_network_isolation.png)
*(Bild: Visualisierung der Netzwerkisolierung in Docker)*

### 3. Disaster Recovery Protokoll (Lessons Learned)
Wir simulierten einen totalen Datenverlust (`docker compose down -v`) und etablierten ein deterministisches Protokoll zur Wiederherstellung.

**Das verifizierte Protokoll:**
```bash
# 1. Verifiziertes Backup erstellen (Echter SQL-Dump)
docker exec vault_db mysqldump -u wp_user -pwp_password_secreto wordpress_db > backup.sql

# 2. Deterministische Wiederherstellung
cat backup.sql | docker exec -i vault_db mariadb -u wp_user -pwp_password_secreto wordpress_db
```

---

## ☸️ Phase 4: K3s Multi-Node Cluster (High Availability)

Um die Infrastruktur ausfallsicher zu machen, skalierten wir auf ein **verteiltes System** und bündelten die Rechenleistung beider Raspberry Pis mittels K3s.

### 🏛️ Cluster-Architektur: Master & Worker

* **Control Plane (Master - `pi5-node-01`):** Verantwortlich für API-Management, Scheduling und den Cluster-Zustand.
* **Worker Node (Agent - `pi4-node-01`):** Führt die eigentlichen Anwendungs-Container aus.

![Foto oder Diagramm der physischen Setup: Pi 5 und Pi 4 verbunden durch einen Switch](./images/k3s_cluster_architecture.png)
*(Bild: Architektur des K3s Clusters)*

### 🛠️ Implementierungsschritte
1. **Kernel-Optimierung (Cgroups):** Kubernetes benötigt eine strikte Ressourcenkontrolle. Anpassung der `/boot/firmware/cmdline.txt` (Aktivierung von `cgroup_memory` und `cpuset`).
2. **Initialisierung:** Installation des Masters via K3s-Skript und Extraktion des Security-Tokens in `/var/lib/rancher/k3s/server/node-token`.
3. **Integration des Workers:** Beitritt des Pi 4 zum Cluster über die Master-IP (`192.168.0.20:6443`) und den generierten Token.

> ⚠️ **Troubleshooting (Exec Format Error):** Beim ersten Deployment-Test stießen wir auf einen Architektur-Fehler. Das Image war nicht für die 64-Bit ARM-Architektur optimiert. **Lösung:** Umstellung auf Multi-Arch-Images.

---

## 🚀 Phase 5: Die große Migration & Production-Ready Setup

Das finale Ziel war die Migration von "El Banco" aus Docker Compose in das Kubernetes-Cluster und die Härtung des Systems für den Produktionseinsatz.

### 5.1 Die Basis: Deployments & Services
Wir teilten den Monolithen in Microservices auf:
* **Deployment:** Der "Manager", der sicherstellt, dass die gewünschte Anzahl an Pods läuft.
* **Service:** Der "Netzwerk-Router", der statische IPs (ClusterIP) oder externe Zugänge (NodePort 30082) zuweist.

### 💥 Der Chaos-Test (Self-Healing-Validierung)
Um die Ausfallsicherheit zu beweisen, löschten wir absichtlich die aktive Datenbank:
```bash
sudo kubectl delete pod vault-db-deployment-xxxxxx
```
**Ergebnis:** Das Control-Plane erkannte den Ausfall in Millisekunden und erstellte den Pod sofort neu (`Terminating` ➡️ `ContainerCreating` ➡️ `Running`). 

---

### 5.2 Datenpersistenz (Persistent Volumes)
**Das Problem:** Container leiden unter "Amnesie". Stirbt ein Datenbank-Pod, sind die Daten weg.
**Die Lösung:** Wir erstellten einen `PersistentVolumeClaim` (PVC), der als virtuelle Festplatte fungiert und Daten dauerhaft auf dem Host speichert.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vault-db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

### 5.3 Security: K8s Secrets (Die kryptografische Vault)
**Das Problem:** Passwörter standen als Klartext in den YAML-Dateien.
**Die Lösung:** Implementierung von Kubernetes `Secrets`. Zugangsdaten wurden in Base64 verschlüsselt und als Referenz sicher übergeben:

```yaml
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: credenciales-banco
      key: root-pass
```

---

### 5.4 Skalierung & Load Balancing
Um Traffic-Spitzen zu bewältigen, änderten wir exakt einen Parameter (`replicas: 3`) im Frontend-Deployment. Der interne Scheduler verteilte die Last sofort intelligent auf beide physischen Maschinen:

```bash
robert@pi5-node-01:~ $ sudo kubectl get pods -o wide
NAME                                     READY   STATUS    NODE
cashier-wp-deployment-6cb4b845b6-9xz7r   1/1     Running   pi5-node-01
cashier-wp-deployment-6cb4b845b6-b264z   1/1     Running   pi5-node-01
cashier-wp-deployment-6cb4b845b6-cdps9   1/1     Running   pi4-node-01
```

![Screenshot des Terminals mit dem Befehl kubectl get pods -o wide, der die Verteilung zeigt](./images/k8s_load_balancing.png)
*(Bild: Kubernetes Scheduler verteilt Workloads dynamisch über die Nodes)*

---

### 5.5 Cluster-Hygiene (Zombie-Pods entfernen)
Wir lernten, "Zombie-Ressourcen" (Pods, die nach einem Node-Fehler in `Terminating` feststeckten) gewaltsam aus der K8s-Datenbank zu entfernen:
```bash
sudo kubectl delete pod bank-web-5d5f8cb566-svgcq --grace-period=0 --force
```

---

### 5.6 Ingress: Der professionelle Zugang
Anstatt über unhandliche IPs und Ports zuzugreifen, konfigurierten wir einen **Ingress Controller** (Traefik). 
Durch die Anpassung der `/etc/hosts` auf dem Client-Mac routet Traefik den Traffic nun elegant und unsichtbar:

* **URL:** `http://elbanco.local`
* **Flow:** Browser ➡️ Ingress ➡️ Service ➡️ Pod

![Screenshot der fertigen WordPress-Seite im Browser mit der URL elbanco.local](./images/ingress_elbanco_local.png)
*(Bild: Das fertige System, erreichbar über eine lokale Domain)*

---

## 🏁 Fazit
Dieses Projekt demonstriert den erfolgreichen Aufbau einer **Production-Ready Architektur**. Das System heilt sich selbst, speichert Daten persistent ab, verschlüsselt Geheimnisse, skaliert nahtlos über mehrere Hardware-Knoten und routet Traffic über professionelle Ingress-Regeln.