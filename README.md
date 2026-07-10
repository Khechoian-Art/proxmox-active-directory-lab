# High-Availability Active Directory Infrastructure Deployment on Proxmox VE

## 🚀 Project Overview
This project demonstrates the deployment, configuration, and troubleshooting of an enterprise-grade, fault-tolerant infrastructure using **Windows Server 2025** on top of a **Proxmox VE** hypervisor environment. The goal was to build a resilient multi-node Active Directory Domain Services (AD DS) ecosystem, handle real-world hypervisor-level integration challenges, and prepare a baseline for auxiliary services (DHCP, DNS redundancy, LXC Linux workloads).

---

## 🛠️ Tech Stack & Infrastructure Baseline
- **Hypervisor:** Proxmox VE
- **Operating Systems:** Windows Server 2025 (Standard Evaluation), Ubuntu/Debian (planned LXC nodes)
- **Core Services:** Active Directory Domain Services (AD DS), DNS, DHCP Server
- **Hardware Virtualization Layer:** QEMU (Machine Type: `q35`, BIOS: `OVMF UEFI`, TPM 2.0)
- **Storage & Network Drivers:** VirtIO (SCSI pass-through controllers, paravirtualized VirtIO NICs)

---

## 📈 Network & Domain Architecture
- **Internal Domain FQDN:** `labprox61.local`
- **Subnet:** `192.168.17.0/24`

### Node Directory:
| Node Name | Role | IP Address | Primary DNS | Secondary DNS |
| :--- | :--- | :--- | :--- | :--- |
| **DOMC01** | Primary Domain Controller (PDC) | `192.168.17.10` | `127.0.0.1` | `192.168.17.11` |
| **DOMC02** | Secondary Domain Controller (BDC) | `192.168.17.11` | `192.168.17.10` | `127.0.0.1` |

---

## 🔧 Deep-Dive Implementation & Troubleshooting Cases

The value of this home lab lies in resolving complex integration issues that frequently occur when running modern Microsoft OS stacks on open-source hypervisors. Below are the key engineering challenges faced and solved during the deployment.

### 📌 Case 1: Windows Server 2025 UEFI Installation Timeout (Proxmox VirtIO Bug)
- **Problem:** During the initial installation boot sequence with Machine Type `q35` and `OVMF (UEFI)` BIOS, the installer would drop into an infinite loop or throw a UEFI Shell timeout error when using the standard VirtIO SCSI or SATA buses for the installation ISO.
- **Root Cause:** In certain Proxmox configurations, Windows Server 2025 UEFI bootloaders fail to initialize early SATA/SCSI storage layers for the boot CD/DVD before the timeout expires.
- **Resolution:** 1. Set the primary OS storage drive to **SCSI (0)** utilizing VirtIO SCSI pass-through.
  2. Isolated the installation ISO by moving its controller exclusively to the **IDE bus** (`ide2`).
  3. Placed the VirtIO drivers ISO onto a secondary **SATA controller**. 
  4. Adjusted the **Boot Order** in Proxmox Options to prioritize the IDE drive, bypassing the UEFI initialization bug.

### 📌 Case 2: Post-Boot Bootloop & CPU Architecture Incompatibility
- **Problem:** Immediately after passing the UEFI bootloader stage, the virtual machine experienced instant, unlogged reboots (bootloops).
- **Root Cause:** Proxmox defaults to the `kvm64` virtual CPU profile for maximum cross-node migration compatibility. However, Windows Server 2025 relies on modern execution instructions missing from the legacy `kvm64` mask.
- **Resolution:** Modified the CPU Hardware settings, switching the Processor Type from `Default (kvm64)` to **`host`**. This allowed the VM to directly leverage the physical instruction set of the host CPU, immediately stabilizing the system.

### 📌 Case 3: NetBIOS Context Failure & Domain Join Blockers
- **Problem:** When attempting to promote the secondary controller (`DOMC02`) and join it to the existing `labprox61.local` directory, Active Directory threw a critical network resolution warning stating that the DNS server could not be located, despite successful ICMP responses between nodes.
- **Root Cause:** 1. The local machine had an external public DNS (`8.8.8.8`) specified as an alternate adapter setting, causing Windows to poll public root hints instead of querying the Primary Domain Controller for local directory SRV records.
  2. Sub-optimal credential mapping during domain authentication due to localized NetBIOS naming conflicts.
- **Resolution:**
  1. Purged the public `8.8.8.8` address from the network adapter settings, locking down the DNS loop explicitly between `192.168.17.10` and `127.0.0.1`.
  2. Flushed the local resolver resolver cache using PowerShell:
     ```powershell
     ipconfig /flushdns
     ```
  3. Forced the domain join utilizing the explicit NetBIOS security context format: `labprox61\Administrator` instead of plain localized accounts.

---

## 🎯 Current Project Status & Next Steps
1. **Active Directory Replication:** Successfully implemented. Both `DOMC01` and `DOMC02` are running Active Directory Domain Services with verified cross-replication of objects and active DNS zones.
2. **DHCP Integration:** Installed the DHCP Server role on the secondary controller (`DOMC02`) to begin configuring redundant network address provisioning.
3. **Infrastructure Expansion:** The next phase involves spinning up isolated LXC containers within Proxmox to serve as Linux application nodes joined to the network architecture, followed by setting up fine-grained Group Policy Objects (GPOs) for access control.

---
*Developed as part of a hands-on Systems Administration & Hypervisor Engineering portfolio.*
