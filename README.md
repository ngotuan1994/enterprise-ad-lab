# Enterprise Virtualized Hybrid Infrastructure & Active Directory Lab

![Proxmox](https://img.shields.io/badge/Hypervisor-Proxmox_VE-E57000?style=flat&logo=proxmox)
![OPNsense](https://img.shields.io/badge/Firewall-OPNsense-D94A38?style=flat&logo=opnsense)
![Windows Server](https://img.shields.io/badge/Identity-Active_Directory_DS-0078D6?style=flat&logo=windows)
![Networking](https://img.shields.io/badge/Networking-802.1Q_VLANs-blue)

A multi-tiered enterprise homelab built on enterprise server hardware to simulate enterprise network segmentation, identity access management (IAM), systems administration, and endpoint lifecycle management.

---

## 🏗️ Architecture Overview

> 📄 *For the standalone document and breakdown, see [diagram.md](diagram.md).*

```mermaid
flowchart TD
    subgraph PHY ["Physical Layer"]
        GL["GL.iNet Gateway Router<br>192.168.8.1"] -->|Physical Link| ETH["Dell R730 - eno1"]
    end

    subgraph PVE ["Proxmox VE Hypervisor"]
        ETH --> VMBR0["Bridge: vmbr0<br>(WAN Uplink)"]
        VMBR0 --> OPN_WAN["OPNsense WAN<br>vtnet0: 192.168.8.177"]
        
        OPN_LAN["OPNsense LAN Trunk<br>vtnet1"] --> VMBR1["Bridge: vmbr1<br>(VLAN-Aware Virtual Switch)"]
        
        OPN_WAN <-->|Inter-VLAN Routing & NAT| OPN_LAN
    end

    subgraph V20 ["VLAN 20: SERVERS (192.168.20.0/24)"]
        VMBR1 -.->|Tag 20| DC01["DC01: Primary AD DS / DNS<br>192.168.20.10"]
        VMBR1 -.->|Tag 20| DC02["DC02: Secondary AD DS / DNS<br>192.168.20.11"]
        VMBR1 -.->|Tag 20| MECM["MECM01: Systems Management<br>192.168.20.30"]
        
        DC01 <===>|AD Replication / DNS Sync| DC02
    end

    subgraph V30 ["VLAN 30: CLIENTS (192.168.30.0/24)"]
        VMBR1 -.->|Tag 30| WS01["W10-FIN-01: Domain Client<br>192.168.30.50"]
        VMBR1 -.->|Tag 30| WS02["W10-IT-01: Admin Client<br>192.168.30.51"]
    end

    WS01 -->|DNS / Kerberos Auth| DC01
    WS01 -->|Patches & Deployment| MECM
```

---

## 🌐 Network Segmentation Plan

| VLAN ID | Subnet | Name | Description |
| :--- | :--- | :--- | :--- |
| **VLAN 1** | `192.168.8.0/24` | **WAN / Uplink** | Physical gateway network connected via GL.iNet router |
| **VLAN 10** | `192.168.10.0/24`| **MGMT** | Out-of-band hypervisor & infrastructure management |
| **VLAN 20** | `192.168.20.0/24`| **SERVERS** | Tier-0 Core Services (Active Directory, DNS, MECM) |
| **VLAN 30** | `192.168.30.0/24`| **CLIENTS** | Enterprise client workstations & test environments |
| **VLAN 40** | `192.168.40.0/24`| **SEC / SOC** | Monitoring, SIEM (Security Onion / Wazuh), & IDS |

---

## 💻 Infrastructure Hosts & Roles

| Hostname | Role | IP Address | OS / Specs |
| :--- | :--- | :--- | :--- |
| **`OPNsense-FW`** | Core Gateway / Routing / DHCP | `192.168.20.1`, `192.168.30.1` | FreeBSD / OPNsense |
| **`DC01`** | Primary Domain Controller / DNS / FSMO | `192.168.20.10` | Windows Server 2022 |
| **`DC02`** | Secondary DC / Redundant DNS / HA | `192.168.20.11` | Windows Server 2022 |
| **`MECM01`** | Endpoint Management / WSUS / SQL | `192.168.20.30` | Windows Server 2022 |
| **`W10-FIN-01`** | Domain-Joined Workstation | `192.168.30.50` (DHCP) | Windows 10/11 Enterprise |

---

## ⚙️ Key Technical Implementations

### 1. Layer 2/3 Virtual Networking & Routing
- Configured Linux bridge `vmbr1` on Proxmox with **802.1Q VLAN awareness** (`bridge-vlan-aware yes`).
- Implemented virtual trunking into OPNsense with isolated sub-interfaces (`vlan0.20`, `vlan0.30`).
- Configured dynamic address allocation via **Kea DHCP / ISC DHCP** with custom DNS options pointing to internal Domain Controllers.
- Configured Hybrid Outbound NAT and strict inter-VLAN firewall rules allowing necessary Active Directory RPC, Kerberos, DNS, and LDAP ports.

### 2. Active Directory Domain Services (AD DS)
- Deployed a multi-domain forest (`lab.local`) with hierarchical Organizational Unit (OU) structure following enterprise RBAC standards.
- Configured active multi-master AD DS replication and integrated DNS zones with secondary resolvers.
- Automated user lifecycle onboarding, group memberships, and account unlock procedures using PowerShell.

### 3. Group Policy Architecture
- Centralized security configurations using Group Policy Objects (GPOs):
  - Standardized mapped network drives (`Z:\SharedFiles`) targeted via Security Group Filtering.
  - Windows Defender firewall policy enforcement and local administrator password restrictions (LAPS).
  - Software Update Point redirection targeting internal WSUS/MECM infrastructure.

---

## 🛠️ Verification & Troubleshooting Runbook

### Active Directory Health Checks
```powershell
# Verify domain controller replication
repadmin /replsummary
repadmin /showrepl

# Verify active FSMO role holders
netdom query fsmo

# Test secure channel from client workstation
Test-ComputerSecureChannel -Verbose
```

### Client Diagnostics & Group Policy
```cmd
:: Force GPO policy synchronization
gpupdate /force

:: Generate GPO diagnostic results summary
gpresult /r

:: Flush DNS resolver cache
ipconfig /flushdns
```
