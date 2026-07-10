# High-Availability Active Directory Infrastructure & Services on Proxmox VE

## 🚀 Project Overview
This project demonstrates the deployment, configuration, and troubleshooting of an enterprise-grade, fault-tolerant infrastructure. The core is a multi-node Active Directory Domain Services (AD DS) ecosystem built on **Windows Server 2025**, supported by lightweight Linux containers (LXC) for monitoring and disaster recovery, all managed within a **Proxmox VE** hypervisor cluster.

---

## 🛠️ Tech Stack & Infrastructure Baseline
- **Hypervisor:** Proxmox VE (Clustered environment)
- **Virtual Machines (Windows):** Windows Server 2025 (Standard Evaluation) for Core AD services
- **Containers (Linux LXC):** Debian/Ubuntu templates for auxiliary services
- **Core Services:** Active Directory (AD DS), DNS, DHCP Server
- **Infrastructure Services:** Proxmox Backup Server (PBS), Uptime Kuma (Monitoring)
- **Virtualization Layer:** QEMU (Machine Type: `q35`, `OVMF UEFI`), LXC (Linux Containers)

---

## 📈 Network & Architecture Topology
- **Cluster Name:** `LAB-CLUSTER`
- **Internal Domain FQDN:** `labprox61.local`
- **Subnet:** `192.168.17.0/24`

### Virtual Machines & Containers Allocation:
| ID | Hostname | Type | Role / Service | Node Location | IP Address |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **100** | `DOMC01` | VM | Primary Domain Controller (PDC, DNS) | `prox61` | `192.168.17.10` |
| **103** | `DOMC02` | VM | Secondary Domain Controller (BDC, DHCP) | `prox91` | `192.168.17.11` |
| **101** | `monitor` | LXC | Monitoring Dashboard (Uptime Kuma) | `prox61` | DHCP / Static |
| **102** | `pbs` | LXC | Proxmox Backup Server (Disaster Recovery) | `prox61` | DHCP / Static |

*(Вставь сюда скриншот твоего Proxmox, где видно список виртуалок и графики CPU)*

---

## 🔧 Deep-Dive Implementation & Troubleshooting Cases

The value of this home lab lies in resolving complex integration issues and optimizing resource allocation. Below are the key engineering challenges faced during the deployment.

### 📌 Case 1: Windows Server 2025 UEFI Installation Timeout (VirtIO Bug)
- **Problem:** During the initial installation boot sequence with Machine Type `q35` and `OVMF (UEFI)` BIOS, the installer would drop into an infinite loop or throw a UEFI Shell timeout error.
- **Root Cause:** In certain Proxmox configurations, Windows Server 2025 UEFI bootloaders fail to initialize early SATA/SCSI storage layers for the boot CD/DVD.
- **Resolution:** 1. Isolated the installation ISO by moving its controller exclusively to the **IDE bus** (`ide2`).
  2. Placed the VirtIO drivers ISO onto a secondary **SATA controller**. 
  3. Adjusted the **Boot Order** to prioritize the IDE drive, successfully bypassing the UEFI initialization bug.

### 📌 Case 2: Post-Boot Bootloop & CPU Architecture Incompatibility
- **Problem:** Immediately after passing the UEFI bootloader stage, the VM experienced instant, unlogged reboots.
- **Root Cause:** Proxmox defaults to the `kvm64` virtual CPU profile. However, Windows Server 2025 relies on modern execution instructions missing from the legacy `kvm64` mask.
- **Resolution:** Modified the CPU Hardware settings, switching the Processor Type from `Default (kvm64)` to **`host`**. This allowed the VM to directly leverage the physical instruction set of the host CPU.

### 📌 Case 3: NetBIOS Context Failure & Domain Join Blockers
- **Problem:** When attempting to promote the secondary controller (`DOMC02`), Active Directory threw a critical network resolution warning stating that the DNS server could not be located, despite successful ICMP responses.
- **Root Cause:** The local machine had an external public DNS (`8.8.8.8`) specified as an alternate adapter setting, causing Windows to poll public root hints instead of querying the PDC.
- **Resolution:**
  1. Purged the public `8.8.8.8` address, locking down the DNS loop explicitly between `192.168.17.10` and `127.0.0.1`.
  2. Flushed the local resolver cache (`ipconfig /flushdns`).
  3. Forced the domain join utilizing the explicit NetBIOS security context: `labprox61\Administrator`.

### 📌 Case 4: Resource Optimization via LXC (Monitoring & Backups)
- **Implementation:** Instead of deploying heavy, resource-intensive Windows or Linux Virtual Machines for auxiliary services, I utilized **Proxmox LXC (Linux Containers)**.
- **Services Deployed:**
  - **Proxmox Backup Server (PBS):** Configured in container `102` to handle deduplicated, incremental backups of the critical Active Directory VMs, ensuring a robust Disaster Recovery plan.
  - **Uptime Kuma:** Deployed in container `101` to provide real-time monitoring and alerting for ICMP connectivity and DNS resolution across the `labprox61.local` domain.
- **Outcome:** Achieved a highly responsive infrastructure layer with minimal CPU/RAM overhead (e.g., the PBS container idles at ~2% CPU and less than 150MB of RAM).

---

## 🎯 Current Project Status & Next Steps
1. **Active Directory:** Cluster is fully operational. Both `DOMC01` and `DOMC02` are running AD DS with verified cross-replication.
2. **Infrastructure Services:** Backups (PBS) and Monitoring (Uptime Kuma) containers are successfully deployed and running on Node 1 (`prox61`).
3. **Next Phase:** Configuring the DHCP scope on `DOMC02`, setting up DFS (Distributed File System) replication (`DFSR01` and `DFSR02`), and deploying Docker hosts for Apache web services according to the planned topology.

---
*Developed as part of a hands-on Systems Administration & Hypervisor Engineering portfolio.*
