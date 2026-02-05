
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
