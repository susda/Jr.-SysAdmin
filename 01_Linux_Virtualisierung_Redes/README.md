# 🛠️ Fase 1: Linux, Virtualisierung und Redes

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

---

## 🛠️ OS-Installation, SSH-Hardening & Netzwerk-Diagnose

Der erste Schritt bestand darin, die Nodes (Raspberry Pi 4 & 5) mit einem minimalen Betriebssystem zu provisionieren und den sicheren Fernzugriff zu gewährleisten. Dabei kamen fortgeschrittene Diagnose-Tools zum Einsatz.

### 1. Image-Provisionierung (Raspberry Pi Imager)
Für die Erstinstallation wurde **Raspberry Pi OS Lite (64-bit)** gewählt. Über die "OS Customization"-Funktion wurden vorab folgende Parameter injiziert:
* **Hostname:** `pi5-node-01` / `pi4-node-01`
* **SSH:** Aktiviert (Public-Key Authentication only).
* **User:** `robert`
* **Locale:** Europe/Berlin.

### 2. Advanced Connectivity Troubleshooting (Deep Dive)
Während des Wechsels zwischen Boot-Medien (NVMe vs. SD-Karte) traten Konflikte bei der Host-Identifizierung auf. 

#### 🕵️‍♂️ Verbose-Debugging (`ssh -v`)
Anstatt nur auf "Connection Refused" zu starren, wurde der SSH-Client im Verbose-Mode ausgeführt:
```bash
ssh -v robert@192.168.0.20