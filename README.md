# High-Availability Active Directory Infrastructure & Services on Proxmox VE

## 🚀 Project Overview
This project demonstrates the deployment, configuration, and troubleshooting of an enterprise-grade, fault-tolerant infrastructure. The core is a multi-node Active Directory Domain Services (AD DS) ecosystem built on **Windows Server 2025**, supported by lightweight Linux containers (LXC) for monitoring and disaster recovery, all managed within a **Proxmox VE** hypervisor cluster (`LAB-CLUSTER`).

---

## 🛠️ Tech Stack & Infrastructure Baseline
- **Physical Hardware & Virtualization:** Bare-metal Proxmox VE installation, High-Availability Clustering.
- **Network Fabric:** 1 Core Router, 2 Managed L2/L3 Switches (802.1Q VLAN tagging, trunking, inter-switch links).
- **Virtual Machines (Windows):** Windows Server 2025 (Standard Evaluation) for Core AD services.
- **Containers (Linux LXC):** Debian/Ubuntu templates for auxiliary services.
- **Core Services:** Active Directory (AD DS), DNS, DHCP Server.
- **Infrastructure Services:** Proxmox Backup Server (PBS), Uptime Kuma (Monitoring).
- **Virtualization Layer:** QEMU (Machine Type: `q35`, `OVMF UEFI`), LXC (Linux Containers).

---

## 📈 Network & Architecture Topology
- **Cluster Name:** `LAB-CLUSTER`
- **Internal Domain FQDN:** `labprox61.local`
- **Subnet:** `192.168.17.0/24`

### Virtual Machines & Containers Allocation:
| ID | Hostname | Type | Role / Service | Node Location | IP Address |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **100** | `DOMC01` | VM | Primary Domain Controller (PDC, DNS) | `prox61` | Static (`192.168.17.10`) |
| **103** | `DOMC02` | VM | Secondary Domain Controller (BDC, DHCP) | `prox91` | Static (`192.168.17.11`) |
| **101** | `monitor` | LXC | Monitoring Dashboard (Uptime Kuma) | `prox61` | Static (`192.168.17.X`) |
| **102** | `pbs` | LXC | Proxmox Backup Server (Disaster Recovery) | `prox61` | Static (`192.168.17.Y`) |

*(ФОТО 1: Сюда перетащи скриншот дерева Proxmox из веб-интерфейса, где слева видны обе ноды prox61 и prox91 со всеми запущенными машинами 100-103)*

---

## 🔧 Deep-Dive Implementation & Troubleshooting Cases

The value of this home lab lies in resolving complex integration issues and optimizing resource allocation. Below are the key engineering challenges faced during the deployment.

### 📌 Case 0: Bare-Metal Provisioning & Network Fabric Setup
- **Implementation:** Before deploying virtual machines, the underlying physical and network architecture was engineered from scratch.
- **Proxmox Clustering:** Installed Proxmox VE on bare-metal hardware. Configured Linux networking bridges (`vmbr0`) and established a stable, multi-node hypervisor cluster linking `prox61` and `prox91` nodes.

#### 🌐 Network Configuration & Implementation Details
To achieve complete isolation for the domain traffic, the network infrastructure was partitioned into explicit logical segments using a combination of L2 switching, hypervisor-level bridging, and static guest OS routing.

##### Step 1: Physical Network Layer (Router & Dual-Switch Trunking)
- **VLAN ID:** `17` (Dedicated for isolated Active Directory management, replication, and cluster traffic).
- **Trunking Implementation:**
  - The **Core Router** was configured with subinterfaces to handle inter-VLAN routing and provide upstream access.
  - An **802.1Q Trunk link** was established to interconnect **Switch 1** and **Switch 2**, successfully extending the logical layer-2 broadcast domain (VLAN 17) across both physical switches.
  - The physical uplinks from both **Proxmox hosts** (`prox61` and `prox91`) were connected to designated **Trunk ports** on their respective switches. This ensures that tagged VLAN 17 frames are passed natively into the hypervisors' virtual bridges (`vmbr0`) without stripping the VLAN headers.

##### Step 2: Hypervisor Layer (Proxmox VE Network Bridge)
- **Virtual Bridge:** `vmbr0` (Linux Bridge mapped to the physical network interface card).
- **VLAN Aware:** Enabled on `vmbr0` to allow the virtual switch to seamlessly manage multiple VLAN tags.
- **Guest NIC Assignment:** For both domain controllers (`DOMC01` and `DOMC02`), the virtual network interface was linked to `vmbr0` with the **VLAN Tag** field explicitly set to `17`.

##### Step 3: Guest OS Layer (Windows Server Static IP Architecture)
To ensure reliable DNS resolution and prevent domain replication failures, DHCP was bypassed on the infrastructure layer.
- **DOMC01:** IP `192.168.17.10` | Preferred DNS: `127.0.0.1` | Alternate DNS: `192.168.17.11`
- **DOMC02:** IP `192.168.17.11` | Preferred DNS: `192.168.17.10` | Alternate DNS: `127.0.0.1`

---

### 📌 Case 1: Windows Server 2025 UEFI Installation Timeout (VirtIO Bug)
- **Problem:** During the initial installation boot sequence with Machine Type `q35` and `OVMF (UEFI)` BIOS, the installer would drop into an infinite loop or throw a UEFI Shell timeout error.
- **Root Cause:** In certain Proxmox configurations, Windows Server 2025 UEFI bootloaders fail to initialize early SATA/SCSI storage layers for the boot CD/DVD.
- **Resolution:**
  1. Isolated the installation ISO by moving its controller exclusively to the **IDE bus** (`ide2`).
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

*(ФОТО 2: Сюда перетащи скриншот из консоли cmd/PowerShell с успешным пингом ping 192.168.17.10 между контроллерами домена)*

---

### 📌 Case 4: Resource Optimization via LXC (Monitoring & Backups)
- **Implementation:** Instead of deploying heavy, resource-intensive Windows or Linux Virtual Machines for auxiliary services, I utilized **Proxmox LXC (Linux Containers)**.
- **Services Deployed:**
  - **Proxmox Backup Server (PBS):** Configured in container `102` to handle deduplicated, incremental backups of the critical Active Directory VMs, ensuring a robust Disaster Recovery plan.
  - **Uptime Kuma:** Deployed in container `101` to provide real-time monitoring and alerting for ICMP connectivity and DNS resolution across the `labprox61.local` domain.
- **Outcome:** Achieved a highly responsive infrastructure layer with minimal CPU/RAM overhead (the PBS container idles at ~2% CPU and less than 150MB of RAM).

---

## 📄 Hardware Running Configurations & Network Dumps

Below are the verification captures and configuration files extracted directly from the physical network devices and Proxmox configuration layers via CLI.

### 1. Core Router Running Configuration (`show running-config`)
*(ФОТО 3: Сюда перетащи скриншот терминала/консоли с выводом конфигурации твоего роутера, где видны настройки сабинтерфейсов или маршрутов для лабораторной сети)*

### 2. Switch 1 & Switch 2 Interconnect Configuration (`show running-config`)
*(ФОТО 4: Сюда перетащи скриншот консоли со свитча, где виден кусок конфига 'show running-config' с транковыми портами 802.1Q между коммутаторами и в сторону Proxmox)*

### 3. Proxmox VE Node Network Config (`cat /etc/network/interfaces`)
*(ФОТО 5: Сюда перетащи скриншот из консоли Proxmox, где открыт текстовый файл сетевых настроек хоста с мостом vmbr0 и включенным vlan-aware)*

---

## 🎯 Current Project Status & Next Steps
1. **Active Directory:** Cluster is fully operational. Both `DOMC01` and `DOMC02` are running AD DS with verified cross-replication.
*(ФОТО 6: Сюда вставь тот крутой скриншот из Proxmox, где внутри винды открыта оснастка AD Users and Computers и папка Domain Controllers с серверами DOMC01 и DOMC02)*

2. **Infrastructure Services:** Backups (PBS) and Monitoring (Uptime Kuma) containers are successfully deployed with static IP addresses and running on Node 1 (`prox61`).
*(ФОТО 7: Сюда перетащи скриншот Server Manager диспетчера серверов на DOMC02, где в левой колонке горят зеленым роли AD DS, DNS и DHCP)*

3. **Next Phase:** Configuring the DHCP scope on `DOMC02`, setting up DFS (Distributed File System) replication (`DFSR01` and `DFSR02`), and deploying Docker hosts for Apache web services according to the planned topology.

---
*Developed as part of a hands-on Systems Administration & Hypervisor Engineering portfolio.*
