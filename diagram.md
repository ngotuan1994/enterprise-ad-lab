```mermaid
flowchart TD
    subgraph Physical_Layer["Physical Layer"]
        GL["GL.iNet Gateway Router<br/>192.168.8.1"] -->|Physical Link| ETH["Dell R730 - eno1"]
    end

    subgraph Proxmox_Hypervisor["Proxmox VE Hypervisor"]
        ETH --> VMBR0["Proxmox Bridge: vmbr0<br/>(WAN Uplink)"]
        VMBR0 --> OPN_WAN["OPNsense WAN<br/>vtnet0: 192.168.8.177"]
        
        OPN_LAN["OPNsense LAN Trunk<br/>vtnet1"] --> VMBR1["Proxmox Bridge: vmbr1<br/>(VLAN-Aware Virtual Switch)"]
        
        OPN_WAN ---|Routing & NAT| OPN_LAN
    end

    subgraph VLAN20["VLAN 20: SERVERS (192.168.20.0/24)"]
        VMBR1 -.->|Tag 20| DC01["DC01: Primary AD DS / DNS<br/>192.168.20.10"]
        VMBR1 -.->|Tag 20| DC02["DC02: Secondary AD DS / DNS<br/>192.168.20.11"]
        VMBR1 -.->|Tag 20| MECM["MECM01: Systems Management<br/>192.168.20.30"]
        
        DC01 ---|AD Replication & DNS Sync| DC02
    end

    subgraph VLAN30["VLAN 30: CLIENTS (192.168.30.0/24)"]
        VMBR1 -.->|Tag 30| WS01["W10-FIN-01: Domain Client<br/>192.168.30.50"]
        VMBR1 -.->|Tag 30| WS02["W10-IT-01: Admin Client<br/>192.168.30.51"]
    end

    WS01 -->|DNS / Kerberos Auth| DC01
    WS01 -->|Patches & Apps| MECM
