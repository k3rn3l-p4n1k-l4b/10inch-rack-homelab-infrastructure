# Headless Linux Host-Based Firewall & Network ACL Isolation

This project documents the implementation of a host-based firewall using **UFW (Uncomplicated Firewall)** on a headless Linux routing host. The system relies on a dual-layer security posture: local application security is handled on-host via UFW, while total data privacy and an un-tunneled traffic **killswitch** are strictly enforced at the network core via hardware Access Control Lists (ACLs).

## 🛡️ Architecture & Defense-in-Depth Design

By decoupling the VPN leak protection from the host operating system, the infrastructure achieves robust fault tolerance. If the host OS or local rulesets are altered, the network security layer safely quarantines any unencrypted traffic before it can exit the WAN gateway.

### Multi-Tier Rules Strategy:
1.  **Network-Layer ACLs**: Actively monitors traffic states. If un-tunneled outbound traffic trying to bypass the `tun0` endpoint is detected at the switch/router interface, the network instantly drops the packet stream.
2.  **Host-Layer UFW**: Enforces strict inbound isolation to protect open daemons (SSH/Web UIs) while maintaining standard default routing rules natively on the physical interface (`eth0`).

> ![ip addr output](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/ipaddr-screenshot%20jpg.jpg)
---

## 🔧 Active UFW Configuration Profile

The local host-based rules allow granular access for management workstations and local DNS resolution while exposing the necessary application interfaces.

> ![UFW Active Ruleset CLI](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/Verbose-UFW%20jpg.jpg)

---

## 🔧 Core Infrastructure Implementation

### 1. Hardening Management Sockets
To preserve access to the headless machine, explicit workstation whitelist definitions are committed directly to the physical interface profile:
```bash
# Allow dedicated administrative workstations access to shell interfaces
sudo ufw allow in on eth0 to any port 22 proto tcp comment 'Admin PC SSH Access'

# Allow local web interface operations
sudo ufw allow in on eth0 to any port 8080 proto tcp comment 'WebGUI access'
```

### 2. Upstream Network Dependency
Because **Rule 13** explicitly allows local outbound failover (`ALLOW OUT Anywhere`), network privacy relies wholly on upstream hardware definitions. In an enterprise scenario, this topology ensures that if local application configurations diverge, the centralized network security engine maintains full control.

## 🎛️ Upstream Network Dependency & Gateway Killswitch

While the local host firewall manages internal application accessibility, absolute WAN privacy is governed upstream at the network layer. This infrastructure utilizes an  **SDN Controller** to implement a hardware-level network killswitch, guaranteeing data isolation even if local operating system configurations diverge.

```text
                  ┌──────────────────────────────┐
                  │  Headless Linux Host │
                  │     (UFW: Default Outbound)  │
                  └──────────────┬───────────────┘
                                 │
                 [Raw Traffic Leaves Interface]
                                 │
                                 ▼
                  ┌──────────────────────────────┐
                  │         SDN Gateway          │
                  │  (Evaluates Gateway ACLs)    │
                  └──────────────┬───────────────┘
                                 │
        ┌────────────────────────┴────────────────────────┐
        ▼                                                 ▼
[Is traffic tunneled?]                            [Is traffic raw WAN?]
        │                                                 │
        ▼                                                 ▼
 ✅ ALLOWED OUT TO INTERNET                        ❌ DENIED / KILLED INSTANTLY
```

### 1. Hardware Access Control List (ACL) Design
A **Gateway ACL** rule is bound to the server's specific IP/VLAN segment. The policy evaluates traffic states at the router interface before routing packets to the WAN gateway:
*   **Direction**: `LAN -> WAN`
*   **Policy**: `Deny`
*   **Source**: Network IP Group (e.g., `XXX.XXX.X.X/24` or the host's dedicated IP)
*   **Destination**: `IP Group Any` (Public Internet)
*   **Execution Protocol**: If the host attempts to pass un-tunneled traffic natively out of its interface to the internet, the Gateway drops the packet sequence instantly. The only traffic allowed to pass the gateway is the established encrypted tunnel protocol to the verified VPN destination endpoint IP.

> ![TP-Link Omada Gateway ACL Killswitch](https://github.com/k3rn3l-p4n1k-l4b/10inch-rack-homelab-infrastructure/blob/main/images/SDN%20rule.jpg)

### 2. Architectural Redundancy Benefit
By utilizing this topology, the infrastructure avoids single-point-of-failure liabilities. If a local configuration change or software crash accidentally alters the host's default fallback routing posture, the upstream network boundary provides automated, transparent fail-safe containment.

