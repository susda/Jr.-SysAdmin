# 🚀 Homelab Projekt: Von Docker zu Kubernetes (K3s)

Dieses Repository dokumentiert den Aufbau einer stabilen, sicheren und skalierbaren containerbasierten Infrastruktur. Das Projekt begann mit **Docker & Portainer** (Phase 3) und entwickelte sich zu einem voll funktionsfähigen **High-Availability K3s Kubernetes Cluster** (Phase 4 & 5) auf einem Raspberry Pi 5 (Master) und Pi 4 (Worker).

Der Fokus liegt auf den Prinzipien von *Infrastructure as Code* (IaC), Netzwerkisolierung, *Disaster Recovery* und moderner Microservice-Orchestrierung.

---

## 🎯 Projektziele & Meilensteine

- ✅ **Imperativ zu Deklarativ:** Migration von manuellen Befehlen zu YAML-Definitionen (IaC).
- ✅ **Netzwerkisolierung (Security):** Isolierte Backend-Netzwerke zum Schutz sensibler Microservices.
- ✅ **Datenpersistenz:** Sicherstellung, dass kritische Daten Neustarts und Pod-Zerstörungen überleben.
- ✅ **Disaster Recovery Protokoll:** Etablierung verifizierter Backup- und Wiederherstellungsverfahren.
- ✅ **Multi-Node Orchestrierung:** Verteilung von Workloads über mehrere physische Maschinen (K3s).
- ✅ **Secret Management:** Verschlüsselung von Zugangsdaten nach Industriestandard.
- ✅ **Load Balancing & Ingress:** Bereitstellung der Services über professionelle Domains (z.B. `elbanco.local`).

---

# 🐳 Phase 3: Single-Node Docker & Disaster Recovery

Im ersten Schritt haben wir die Laufzeitumgebung stabilisiert und eine komplexe Webanwendung („Das Bankhaus“) auf einem einzigen Knoten aufgebaut.

### 1.1 Container Runtime & Identity Management
Bevor wir Dienste bereitstellen konnten, mussten wir die logische Identität des Pi 5 Knoten konfigurieren:
* **Identität:** Etablierung einer logischen Nomenklatur (`pi5-node-01`) via `/etc/hosts` und `hostnamectl`.
* **Installation:** Bereitstellung des offiziellen Docker GPG Keys und Installation der Docker Engine & Docker Compose.

### 1.2 "Das Bankhaus": Netzwerkisolierung & Sicherheit
Wir deployten eine Multi-Container-Anwendung (WordPress + MariaDB), um den Datenfluss zu demonstrieren.

* **Der Kassierer (Frontend - WordPress):** Exponiert in das lokale Heimnetzwerk (Port 8082).
* **Der Tresor (Backend - MariaDB):** Kommuniziert **ausschließlich** intern im isolierten `bank_net` Netzwerk auf Port 3306. 

> 🛡️ **Security Check:** Die Datenbank hat keine offenen Ports nach außen. Die Angriffsfläche ist drastisch reduziert.

### 1.3 Disaster Recovery Protokoll (Lessons Learned)
Wir simulierten einen totalen Datenverlust (`docker compose down -v`) und etablierten ein verifiziertes Protokoll.

**Das verifizierte Protokoll:**
```bash
# 1. Verifiziertes Backup erstellen
docker exec vault_db mysqldump -u wp_user -pwp_password_secreto wordpress_db > backup.sql

# 2. Deterministische Wiederherstellung
cat backup.sql | docker exec -i vault_db mariadb -u wp_user -pwp_password_secreto wordpress_db


☸️ Phase 4: K3s Multi-Node Cluster (High Availability)
Nach der erfolgreichen Implementierung von Docker haben wir die Infrastruktur auf ein verteiltes System skaliert. Wir nutzen K3s (Lightweight Kubernetes), um die Rechenleistung beider Raspberry Pis zu bündeln.

🏛️ Cluster-Architektur: Master & Worker
Control Plane (Master - pi5-node-01): Verantwortlich für API-Management, Scheduling und den Cluster-Zustand.

Worker Node (Agent - pi4-node-01): Führt die eigentlichen Anwendungs-Container (Pods) aus.

(Visualisierung der Cluster-Architektur)

🛠️ Implementierungsschritte
Kernel-Optimierung (Cgroups): Aktivierung von cgroup_memory und cpuset in /boot/firmware/cmdline.txt für strikte Ressourcenkontrolle.

Initialisierung: Installation des Masters via K3s-Skript und Extraktion des Security-Tokens.

Integration des Workers: Beitritt des Pi 4 zum Cluster über die Master-IP und den Token.

🔍 Troubleshooting (Exec Format Error): Beim ersten Test stießen wir auf einen Architektur-Fehler. Das Image war nicht für ARM64 optimiert. Lösung: Umstellung auf Multi-Arch-Images.

🚀 Phase 5: Die große Migration & Production-Ready Setup
Das Hauptziel dieses Meilensteins war die Migration von "El Banco" in das Kubernetes-Cluster und die Härtung des Systems für den Produktionseinsatz.

5.1 Die Basis: Deployments & Services
Wir haben die monolithische Struktur in Microservices aufgeteilt:

Deployment: Der "Manager", der sicherstellt, dass die Pods laufen.

Service: Der "Netzwerk-Router", der statische IPs (ClusterIP) oder externe Zugänge (NodePort 30082) zuweist.

💥 Der Chaos-Test (Self-Healing-Validierung)
Um die Ausfallsicherheit zu beweisen, löschten wir absichtlich einen aktiven Pod:

sudo kubectl delete pod <pod-name>
Ergebnis: Das Control-Plane erkannte den Ausfall in Millisekunden und erstellte den Pod sofort neu. Die Anwendung blieb ununterbrochen erreichbar.

5.2 Datenpersistenz (Persistent Volumes)
Das Problem: Container leiden unter "Amnesie". Stirbt ein Datenbank-Pod, sind alle Benutzerdaten verloren.Die Lösung: Trennung von Rechenleistung und Speicherplatz. Wir erstellten einen PersistentVolumeClaim (PVC), der als "virtuelle Festplatte" fungiert und Daten dauerhaft auf der SD-Karte des Hosts speichert.

# Auszug aus db-pvc.yaml
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

Test erfolgreich: Die Datenbank wurde zerstört und neu erstellt. Alle WordPress-Daten blieben durch den PVC vollständig erhalten.


5.3 Security: K8s Secrets (Die kryptografische Vault)
Das Problem: Passwörter standen als Klartext in den YAML-Dateien (ein massives Sicherheitsrisiko in GitHub).Die Lösung: Implementierung von Kubernetes Secrets. Die Passwörter wurden in Base64 verschlüsselt und als Referenz (valueFrom: secretKeyRef) an die Pods übergeben.
# Sichere Übergabe von Credentials
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: credenciales-banco
      key: root-pass

5.4 Skalierung & Load Balancing
Um hohen Traffic zu bewältigen, haben wir die Anwendung skaliert. Durch das Ändern eines einzigen Parameters (replicas: 3 im wp-deployment.yaml) multiplizierte Kubernetes unsere WordPress-Instanzen.

Der interne Scheduler verteilte die Last intelligent auf unsere physischen Maschinen:
robert@pi5-node-01:~/k8s-banco $ sudo kubectl get pods -o wide
NAME                                     READY   STATUS    NODE
cashier-wp-deployment-6cb4b845b6-9xz7r   1/1     Running   pi5-node-01
cashier-wp-deployment-6cb4b845b6-b264z   1/1     Running   pi5-node-01
cashier-wp-deployment-6cb4b845b6-cdps9   1/1     Running   pi4-node-01

WordPress läuft nun gleichzeitig auf dem Pi 5 und dem Pi 4. Der K8s Service agiert als unsichtbarer Load Balancer.

5.5 Cluster-Hygiene (Zombie-Pods entfernen)
Während der Experimente blieben alte Pods in einem endlosen Terminating Status auf nicht mehr existierenden Nodes stecken. Wir haben gelernt, wie man diese Zombie-Ressourcen gewaltsam aus der K8s-Datenbank entfernt:
sudo kubectl delete pod <pod-name> --grace-period=0 --force

5.6 Ingress: Der professionelle Zugang
Anstatt über unhandliche Ports (http://192.168.0.20:30082) auf die Website zuzugreifen, haben wir einen Ingress Controller (Traefik) konfiguriert.

Wir leiteten den Traffic auf eine saubere URL um:

Host: elbanco.local

DNS Trick: Anpassung der /etc/hosts Datei auf dem MacBook, um die URL lokal aufzulösen.

(Traffic-Routing über Traefik Ingress)
📈 Fazit
Dieses Projekt demonstriert den erfolgreichen Aufbau einer Production-Ready Architektur. Das System heilt sich selbst, speichert Daten persistent ab, verschlüsselt Geheimnisse, skaliert über mehrere physische Raspberry Pis und routet Traffic über professionelle Ingress-Regeln.