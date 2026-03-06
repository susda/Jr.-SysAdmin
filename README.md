# 🚀 SysAdmin & Cloud Ops Journey: Vom Homelab zur Produktion

Willkommen in meinem Dokumentations-Repository. 
Dieses Projekt dokumentiert meinen intensiven 8-Wochen-Pfad zum **Junior Systemadministrator / Cloud Operator** in der Rhein-Main-Region.

Das Ziel ist nicht nur das "Ausprobieren" von Software, sondern der Aufbau einer **professionellen, hybriden Infrastruktur**, die reale Unternehmensanforderungen simuliert (Sicherheit, Monitoring, Backups, IaC und CI/CD).

---

## 🗺️ Der Fahrplan (Projekt-Roadmap)

Dieses Repository ist modular aufgebaut. Klicke auf die jeweiligen Phasen, um die detaillierte technische Dokumentation, Architektur-Entscheidungen und Troubleshooting-Guides zu lesen.

### 📁 [Fase 1: Linux, Virtualisierung und Redes](./01_Linux_Virtualisierung_Redes/README.md) *(Semanas 1 y 2)*
* Administration von Linux (User-Management, SSH-Hardening).
* Bare-Metal Installation von **Proxmox VE**.
* VM-Provisionierung (Ubuntu, RHEL/SUSE) mit LVM und Snapshots.
* Layer-2 Netzwerk-Segmentierung, DNS (Pi-hole), DHCP und Firewall (ufw).

### 📁 [Fase 2: Windows Server y Active Directory](./02_Windows_Server_AD/README.md) *(Semana 3)*
* Windows Server 2022 Deployment.
* Active Directory (AD), DNS, OUs und Group Policies (GPO).
* Integration von Linux-Umgebungen in AD via SSSD/realmd.

### 📍 [Fase 3: Docker y Kubernetes](./03_Docker_Kubernetes/README.md) *(Semanas 4 y 5) - CURRENT*
* Container Runtime & Identity Management.
* Portainer Mini-Cluster Setup (Manager & Worker Node).
* Infrastructure as Code (IaC) con Docker Compose / Stacks.
* Implementación de un clúster Kubernetes k3s usando la Raspberry Pi.

### 🚧 [Fase 4: Cloud, IaC y CI/CD](./04_Cloud_IaC_CICD/README.md) *(Semanas 6 y 7)*
* Microsoft Azure (AZ-900 Fundamentos).
* Terraform para Infraestructura como Código (IaC).
* Pipelines CI/CD utilizando GitHub Actions oder GitLab CI.