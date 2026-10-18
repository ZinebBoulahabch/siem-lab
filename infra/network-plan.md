# Network Addressing Plan - SOC Lab (pfSense)

This document outlines the topology of interfaces and subnets configured on the pfSense firewall for the SOC laboratory environment.

## Interface Mapping Table

| pfSense Interface | Zone / VMware Virtual Network | IP Address / Subnet | Zone Role |
| :--- | :--- | :--- | :--- |
| **`em0`** | WAN (VMnet8 - NAT) | `192.168.238.144/24` (DHCP/Static) | Internet Egress / Host Access |
| **`em1`** | LAN (VMnet2) | `10.10.10.1/24` | Main Internal Network / Clients |
| **`em2`** | MGMT (VMnet3) | `10.10.20.1/24` | Management Network (Wazuh Server, Admin) |
| **`em3`** | DMZ (VMnet4) | `10.10.30.1/24` | Demilitarized Zone (Exposed Servers) |
| **`em4`** | ATTACKER (VMnet5) | `10.10.99.1/24` | Attacker Network (Kali Linux) |

## Notes
- Centralized Syslog target server for collection: `10.10.20.10` (MGMT network).
- Inter-zone firewall filtering rules are applied to isolate attack simulation traffic.
