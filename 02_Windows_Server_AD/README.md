
# 🏢 Fase 2: Windows Server y Active Directory (Semana 3)

*📍 Hinweis: Diese Phase befindet sich aktuell in Vorbereitung. Die praktische Dokumentation wird hier nach Abschluss der Labs veröffentlicht.*

## 🎯 Modul-Ziele (Roadmap)

In diesem Modul liegt der Fokus auf der Implementierung von Microsoft-Infrastruktur-Standards, die in Unternehmensumgebungen zwingend erforderlich sind:

1. **Windows Server 2022:**
   * Installation und Härtung von Windows Server 2022 in einer VM auf dem Proxmox-Hypervisor.
2. **Active Directory (AD) & DNS:**
   * Konfiguration von Active Directory Domain Services (AD DS).
   * Aufbau der Domänenstruktur, Units (OUs), Usern und Gruppen.
3. **Group Policy Objects (GPO):**
   * Implementierung von Sicherheitsrichtlinien und Zugriffssteuerungen über GPOs.
4. **Cross-Platform Integration:**
   * Integration der bestehenden Linux-Umgebungen (Ubuntu/RHEL) in das Active Directory unter Verwendung von `SSSD` oder `realmd` für ein zentrales Identity Management (SSO).

📡 Phase 2: Netzwerk-Provisionierung, Storage-Setup & Disaster Recovery
Nach der physischen Montage bestand das Ziel darin, eine statische Netzwerkkonfiguration zu etablieren und den NVMe-Boot für den Master-Knoten einzurichten.

1. Definition der Netzwerkarchitektur (Static Networking)
Ziel: Migration von dynamischer DHCP-Zuweisung zu statischer IP-Adressierung.

Gateway: 192.168.0.1

Proxmox (Hypervisor): 192.168.0.10

Raspberry Pi 5 (Master): 192.168.0.20

Raspberry Pi 4 (Worker): 192.168.0.30

Implementierung: Manuelle Konfiguration via nmtui auf jedem Knoten sowie Deaktivierung des DHCP-Managements für die eth0-Schnittstellen.

Layer-2 Cache Flushing (arp)

Bash
sudo arp -d -a
🔑 Host Key Management (ssh-keygen)
Beim Wechsel vom NVMe-System zum Rescue-SD-System warnte SSH vor einem "Man-in-the-Middle"-Angriff.

Bash
ssh-keygen -R 192.168.0.20
2. Post-Mortem: Kritischer Vorfall (Zugriffssperre & NVMe-Boot-Fehler)
Während der initialen Provisionierung des Master-Nodes (Pi 5) auf der NVMe-SSD ging der administrative Zugriff vollständig verloren.

🔴 Das Problem
Das auf der NVMe installierte Betriebssystem lehnte den konfigurierten öffentlichen SSH-Key ab (Permission denied (publickey)). Das System befand sich in einem "Locked Out"-Status.

🔍 Technische Analyse (Root Cause Analysis)
Ursache: Fehler bei der Injektion der authorized_keys oder Konfigurationsfehler während des Flash-Vorgangs.

Erschwerender Faktor: Der Raspberry Pi priorisierte den Boot-Vorgang von der korrupten NVMe.

Physische Limitierung: Es war kein externer NVMe-Adapter verfügbar.

🛠️ Lösungsstrategie: "Rescue System Technique"
Entwurf einer "Physical Bypass"-Strategie, um die Hardware mittels eines sekundären Speichermediums (MicroSD) zu booten.

✅ Technische Umsetzung (Schritt-für-Schritt)
Erstellung eines Rettungsmediums: Flashen einer SD-Karte mit Backup-Credentials.

Modifikation des Bootloaders (EEPROM): Erzwingen der Boot-Priorität via sudo rpi-eeprom-config. Setzen von BOOT_ORDER=0xf1.

Block-Level-Cloning (dd):

Bash
sudo dd if=/dev/mmcblk0 of=/dev/nvme0n1 bs=4M status=progress
Wiederherstellung der Boot-Reihenfolge: Revertierung auf 0xf416.

Dateisystem-Erweiterung: Nutzung von raspi-config zur Vergrößerung der Partitionstabelle.

🎓 Wichtige Erkenntnisse (Key Takeaways)
Bootloader-Management: Verständnis der EEPROM-Konfiguration.

Linux Storage Handling: Einsatz von lsblk, dd.

Netzwerk-Troubleshooting: Diagnose von IP-Konflikten.

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/8e84e304-ab54-45d9-9f2b-f1c488e0fcb7" />