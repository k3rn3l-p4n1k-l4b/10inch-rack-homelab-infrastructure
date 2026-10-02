# Enterprise Networking & Active Directory Infrastructure Lab Deployment

## 🛠️ Project Phase 1: Virtual Infrastructure & Multi-VLAN Core Routing

### 📋 Executive Summary
This project outlines the architectural deployment and validation of a high-density, multi-subnet enterprise environment virtualized inside a **10-inch Proxmox VE hypervisor node**. The infrastructure leverages **OPNsense** for core edge routing, firewall policies, and internal DNS resolution, interfacing with a physical **TP-Link Layer 2 Managed Smart Switch** via an 802.1Q VLAN trunk line. This segment documents the systematic resolution of a Windows Server TCP/IP network layer corruption and edge firewall tuning to successfully establish a functional, portfolio-ready **Active Directory Domain Services (AD DS)** core pipeline over the virtual trunk.

---

### 🗺️ Network Architecture & Topology

![Network Diagram](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/network_diagram.jpg)

#### Subnet Breakdown
* **VLAN 10 (MGMT):** 10.10.10.0/24 — Mapped to physical Switch Ports 1 & 2. Hosts the primary administration terminal, Proxmox GUI, Switch GUI, and OPNsense GUI management frameworks.
* **VLAN 20 (IoT / WLAN):** 10.10.20.0/24 — Mapped to physical Switch Port 4. Connects the GL.iNet router operating in Access Point (AP) mode to provide isolated wireless services.
* **VLAN 30 (Penetration Testing):** 10.10.30.0/24 — Mapped to physical Switch Port 3. Tied directly to the secondary network interface card (NIC) of the Parrot OS machine to simulate adversary behaviors and engage in red teaming assessments.
* **VLAN 40 (CORP_LAB):** 10.10.40.0/24 — Access mapped via the main vmbr1 Proxmox Trunk. Houses the Windows Active Directory Domain Controllers and Core Identity Management infrastructure.

---

### 🚨 Post-Mortem: Incident Reports & Technical Troubleshooting

#### Incident 1: Windows Server Parameter Parsing Fault (General.77)
* **Symptom:** Active Directory forest deployment failed repeatedly via Server Manager and direct PowerShell cmdlets (Install-WindowsFeature, Install-ADDSForest), yielding fatal error string Test.VerifyDcPromoCore.DCPromo.General.77.
* **Root Cause Analysis (RCA):** A local Windows Remote Management (WinRM) HTTP listener corruption broke the internal communication pipeline used by Server Manager to pass deployment arguments. Concurrently, running aggressive diagnostic table flushes (route /f) wiped the local loopback and subnet routing definitions out of the OS memory entirely, throwing immediate Transmit Failed. General Failure codes inside the TCP/IP stack.
* **Engineering Resolution:** Refreshed the native protocols using low-level initialization hooks (`netsh int ip reset` and `netsh winsock reset`). Bypassed the corrupted scripting parsers by feeding a strict string-literal key payload directly to the legacy core deployment engine via a local text answer file (`dcpromo /answer:C:\adds.txt`). The system cleanly processed the forest generation, bypassed the WinRM block, and executed a successful schema generation.

#### Incident 2: Core Gateway Routing & Unbound DNS Isolation
* **Symptom:** Post-promotion server reboot successfully verified local subnet interface loops (ping 10.10.40.1), but external public routing dead-ended (ping 8.8.8.8 and ping google.com failed).
* **Root Cause Analysis (RCA):** Freshly spawned OPNsense VLAN interfaces strictly enforce default-deny firewall boundaries and lack automatic Network Address Translation rules. Additionally, the internal Unbound DNS daemon lacked explicit listening hooks on the new .40 interface, dropping port 53 packets.
* **Engineering Resolution:** Added three production-grade structural changes inside the OPNsense core configuration:
  1. **Firewall Rules:** Injected an explicit pass rule targeting TCP/UDP on Port 53 (DNS) bound directly to the top of the interface rule matrix.
  2. **Unbound Access Control Lists:** Whitelisted the 10.10.40.0/24 network within the DNS Access Control parameters to authorize local subnet lookups.
  3. **Source NAT Adjustment:** Migrated the global firewall mapping architecture from Automatic to Hybrid Source NAT Mode. Spawned an explicit outbound mapping rule translating private 10.10.40.0/24 subnet requests into the edge WAN interface address.

---

### 🏁 Current State Verification
Following structural network alignment, the domain controller successfully handles external routes. Running an end-to-end trace yields perfect metrics:

* `ping 10.10.40.1` -> Successful (Local Edge Interface Validation)
* `ping 8.8.8.8` -> Successful (Outbound NAT / WAN Edge Validation)
* `ping google.com` -> Successful (Unbound DNS Port 53 Resolution Loop Validation)

The environment stands fully verified, secure from external public vectors, and structurally isolated from neighboring subnets—primed for internal corporate layout development.

#### Active Directory Verification Proof

![Active Directory Verification](images/WindowsActiveDirectory.png)
