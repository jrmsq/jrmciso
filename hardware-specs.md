# Hardware Specifications

## Physical Hardware

### GEEKOM AX8 Max Mini PC

**Why This Hardware?**
- **Quiet Operation**: Fanless or whisper-quiet cooling ideal for home environment
- **Low Power Draw**: Energy-efficient AMD Ryzen architecture
- **High Performance**: 8-core/16-thread CPU handles multiple VMs simultaneously
- **Dual NICs**: Perfect for firewall/router configurations and network monitoring

**Specifications:**
- **CPU**: AMD Ryzen 7 8745HS (8 Cores / 16 Threads)
- **RAM**: 32GB DDR5 (Upgradable for future expansion)
- **Storage**: 1TB NVMe SSD (Expandable)
- **Graphics**: AMD Radeon 780M integrated
- **Network**: Dual 2.5GbE LAN Ports
- **Connectivity**: USB4.0 (40Gbps), multiple USB ports
- **OS**: Windows 11 Pro (will run Proxmox VE)

**Power Consumption**: ~45-65W under load (estimated)

---

## Network Infrastructure

### Managed Switch
- VLAN tagging and trunking support
- Connects all wired devices
- Trunk port to GEEKOM for all VLANs

### TP-Link EAP650 WiFi 6 Access Point
- WiFi 6 (802.11ax) support
- VLAN tagging for wireless networks
- Separate SSIDs per VLAN:
  - Guest WiFi → VLAN 10
  - Home WiFi → VLAN 50
  - IoT WiFi → VLAN 40

### ISP Modem/Router
- WAN connection
- Connected to OPNsense for internet routing

---

## VLAN Architecture

| VLAN ID | Name | Purpose | Subnet (Example) |
|---------|------|---------|------------------|
| **10** | Guest | Visitor devices, untrusted | 192.168.10.0/24 |
| **20** | Management | Proxmox, switches, APs | 192.168.20.0/24 |
| **30** | Lab | Testing, vulnerable VMs | 192.168.30.0/24 |
| **40** | IoT | Smart home devices | 192.168.40.0/24 |
| **50** | Home | Trusted personal devices | 192.168.50.0/24 |

### Inter-VLAN Security Rules
- **Guest VLAN**: Internet only, no access to other VLANs
- **Management VLAN**: Admin access only, restricted by source IP
- **Lab VLAN**: Isolated, no access to Home/IoT
- **IoT VLAN**: Limited internet, no lateral movement
- **Home VLAN**: Full access, controlled outbound rules

---

## Virtual Machine Allocation

### Current VM Inventory

| VM Name | OS | vCPU | RAM | Disk | Primary VLAN | Purpose |
|---------|----|----|-----|------|--------------|---------|
| **OPNsense** | FreeBSD | 2 | 4GB | 32GB | All (Router) | Firewall, DHCP, DNS |
| **SecurityOnion** | Ubuntu | 4 | 16GB | 200GB | VLAN 20 | SIEM/NSM |
| **Windows Server 2022** | Windows | 4 | 8GB | 60GB | VLAN 30 | Active Directory |
| **Windows 11** | Windows | 2 | 4GB | 60GB | VLAN 30 | Endpoint Testing |
| **Ubuntu Server** | Ubuntu | 2 | 4GB | 40GB | VLAN 30 | Linux Services |

**Total Allocated**: 14 vCPUs, 36GB RAM (leaves headroom for additional VMs)

### Future Expansion Options
- Kali Linux (penetration testing)
- Attack simulation VMs (Atomic Red Team, Caldera)
- Additional Windows endpoints for AD domain
- Honeypot systems

---

## Storage Layout

### Proxmox Storage Configuration
```
/dev/nvme0n1
├── /dev/nvme0n1p1  →  EFI Boot (512MB)
├── /dev/nvme0n1p2  →  Proxmox Root (96GB)
└── /dev/nvme0n1p3  →  VM Storage (remaining ~900GB)
    ├── SecurityOnion (200GB)
    ├── Windows Server (60GB)
    ├── Windows 11 (60GB)
    ├── Ubuntu (40GB)
    ├── OPNsense (32GB)
    └── Available (~500GB for future VMs and snapshots)
```

---

## Network Interfaces

### Physical NICs on GEEKOM
- **eth0** (2.5GbE Port 1): WAN connection to ISP
- **eth1** (2.5GbE Port 2): LAN trunk to managed switch (all VLANs tagged)

### OPNsense Virtual Interfaces
- **WAN**: Bridged to eth0
- **LAN**: VLAN trunk on eth1
  - VLAN 10 (Guest)
  - VLAN 20 (Management)
  - VLAN 30 (Lab)
  - VLAN 40 (IoT)
  - VLAN 50 (Home)

### SecurityOnion Network Configuration
- **Management Interface**: VLAN 20
- **Monitoring Interface**: Promiscuous mode, mirrored traffic from switch SPAN port

---

## Monitoring & Data Collection

### Traffic Mirroring
- Switch SPAN/mirror port configured to send all VLAN traffic to SecurityOnion
- Full packet capture available for forensic analysis

### Log Sources
| Source | Log Type | Destination | Protocol |
|--------|----------|-------------|----------|
| OPNsense | Firewall logs | SecurityOnion | Syslog (UDP 514) |
| OPNsense | NetFlow | SecurityOnion | NetFlow v9 |
| Windows endpoints | Sysmon | SecurityOnion | Winlogbeat |
| Windows endpoints | Event logs | SecurityOnion | Winlogbeat |
| Ubuntu | Syslog | SecurityOnion | Rsyslog |

---

## Performance Considerations

### CPU Allocation Strategy
- **Over-provisioning**: 14 vCPUs allocated on 16-thread CPU (safe ratio)
- **CPU pinning**: Not used; allows Proxmox scheduler flexibility
- **Priority VMs**: SecurityOnion gets more CPU shares during contention

### RAM Allocation
- **Total available**: 32GB DDR5
- **Proxmox overhead**: ~2GB
- **VM allocation**: 36GB (over-committed by 6GB)
- **Ballooning enabled**: VMs yield unused memory when needed
- **Swap configured**: 8GB swap for safety

### Disk I/O
- **NVMe SSD**: High-speed storage eliminates bottlenecks
- **VM disk format**: qcow2 with thin provisioning
- **Snapshots**: Enabled for quick rollback during testing

---

## Why This Setup Works

✅ **Single, quiet, low-power device** replaces multiple servers  
✅ **Dual NICs** enable proper firewall/routing architecture  
✅ **32GB RAM** sufficient for 5+ VMs with room to grow  
✅ **Fast NVMe storage** handles SIEM log ingestion without lag  
✅ **5 VLANs** provide realistic enterprise network segmentation  
✅ **Full visibility** via SecurityOnion's network monitoring  

This is not just a lab—it's a production-quality SOC environment at home.
