# Network Addressing & Architecture Plan - SOC Lab (pfSense)

This document details the network topology, IP addressing plan, virtual network mappings, and logical data flow for the SOC Laboratory environment.

---

## 1. Interface & IP Addressing Table

| pfSense Interface | Zone / VMware Network | Gateway / IP Subnet | Zone Role & Security Policy |
| :--- | :--- | :--- | :--- |
| **`em0`** | WAN (VMnet8 - NAT) | `192.168.238.144/24` | Internet Egress / Host Access |
| **`em1`** | LAN (VMnet2) | `10.10.10.1/24` | Internal Corporate Network (AD DS, Clients) |
| **`em2`** | MGMT (VMnet3) | `10.10.20.1/24` | Management Network (SIEM, SOAR, Admin Host) |
| **`em3`** | DMZ (VMnet4) | `10.10.30.1/24` | Demilitarized Zone (Exposed Vulnerable Servers) |
| **`em4`** | ATTACKER (VMnet5) | `10.10.99.1/24` | Isolated Red Team / Attack Simulation Network |

---

## 2. Virtual Machines Inventory & Fixed IP Assignment

| Hostname | Zone / Network | IP Address | OS / Roles | Status / Phase |
| :--- | :--- | :--- | :--- | :--- |
| **pfSense** | All Networks | `.1` on each subnet | FreeBSD - Firewall, DHCP, Suricata IDS | Active (Phase 1) |
| **DC01** | LAN (VMnet2) | `10.10.10.10` | Windows Server 2022 - AD DS, DNS | Active (Phase 2+) |
| **WIN10** | LAN (VMnet2) | `10.10.10.20` | Windows 10 Pro - Enterprise Workstation | Active (Phase 2+) |
| **WAZUH** | MGMT (VMnet3) | `10.10.20.10` | Ubuntu Server - Wazuh Manager, Indexer, Dashboard | Active (Phase 3+) |
| **SOC-TOOLS** | MGMT (VMnet3) | `10.10.20.20` | Ubuntu Server - MISP, Shuffle SOAR, TheHive | Optional (Phase 7) |
| **Host PC** | MGMT (VMnet3) | `10.10.20.254` | Windows Host - Admin Access to Web Consoles | Active |
| **LNX-DMZ** | DMZ (VMnet4) | `10.10.30.10` | Ubuntu Server - Web Server / Attack Target | Active (Phase 2+) |
| **KALI** | ATTACKER (VMnet5) | `10.10.99.10` | Kali Linux - Red Team Attack Box | Active (Phase 5+) |

---

## 3. Infrastructure Diagram

```text
                               ┌────────────────────────┐
                               │     Internet (WAN)     │
                               └───────────┬────────────┘
                                           │
                                   (VMnet8 - NAT)
                                           │
┌──────────────────────────────────────────▼──────────────────────────────────────────┐
│                                   pfSense Firewall                                  │
│                          (Suricata IDS / Syslog Client)                             │
│                                (em0 / WAN: DHCP/NAT)                                │
└───────────────┬──────────────────────┬──────────────────────┬──────────────────────┘
                │                      │                      │                      │
          (em1 / LAN)            (em2 / MGMT)           (em3 / DMZ)         (em4 / ATTACKER)
          VMnet2                 VMnet3                 VMnet4                 VMnet5
       10.10.10.1/24          10.10.20.1/24          10.10.30.1/24          10.10.99.1/24
                │                      │                      │                      │
                │     Agents 1514/1515 │                      │ Inter-zone Rules:    │
                │     Syslog 514/UDP   │                      │ Attacker -> DMZ ONLY │
                │                      │                      │ (Blocked to WAN/MGMT)│
                │                      │                      │                      │
 ┌──────────────┴─────────────┐ ┌──────┴─────────────┐ ┌──────┴─────────────┐ ┌──────┴─────────────┐
 │ DC01                       │ │ WAZUH             │ │ LNX-DMZ             │ │ KALI                │
 │ (10.10.10.10)              │ │ (10.10.20.10)     │ │ (10.10.30.10)       │ │ (10.10.99.10)       │
 │ AD DS / DNS Server         │ │ Server / Indexer  │ │ Web Server / Target │ │ Attacker Machine    │
 └────────────────────────────┘ └───────────────────┘ └─────────────────────┘ └─────────────────────┘
 ┌────────────────────────────┐ ┌ - - - - - - - - - ┐
 │ WIN10                      │ : SOC-TOOLS         :
 │ (10.10.10.20)              │ : (10.10.20.20)     :
 │ Workstation / Agent        │ : MISP, Shuffle,    :
 └────────────────────────────┘ : TheHive (Ph. 7)   :
                                └ - - - - - - - - - ┘
                                       │
                                (VMnet3 / Adapter)
                                       │
                            ┌──────────▼───────────┐
                            │   PC Hôte (Admin)    │
                            │    (10.10.20.254)    │
                            └──────────────────────┘

## 4. Logical Data & Detection Flow

```text
[ Log Sources ]                  [ Collection & Analysis ]       [ Correlation & Dashboard ]           [ Incident Response ]
───────────────                  ─────────────────────────       ───────────────────────────           ─────────────────────

┌─────────────────┐             ┌──────────────────────┐
│ WIN10 / DC01    ├──Sysmon────►│                      │
│ (Wazuh Agents)  │ (1514/1515) │                      │             ┌─────────────────────────┐
└─────────────────┘             │                      │             │ Wazuh Indexer / Dashboard│
                                │                      ├───Index────►│ (Analysis & Dashboards) │
┌─────────────────┐             │ WAZUH Manager        │             └─────────────────────────┘
│ LNX-DMZ         ├──Wazuh─────►│ (10.10.20.10)        │
│ (Wazuh Agent)   │             │                      │             ┌ - - - - - - - - - - - - ┐
└─────────────────┘             │ Decoders / Rules     │◄─Enrich────┤  MISP (Threat Intel)   │
                                │ Mapping MITRE ATT&CK │             │  (10.10.20.20 - Ph. 7)  │
┌─────────────────┐             │                      │             └ - - - - - - - - - - - - ┘
│ pfSense         ├──Syslog────►│                      │
│ (System/Suricata│ (514/UDP)   │                      │
└─────────────────┘             └──────────┬───────────┘
                                           │
                                    (Alert Level >= 10)
                                           │
                                           ▼
                                ┌ - - - - - - - - - - - ┐            ┌ - - - - - - - - - - - - ┐
                                : Shuffle (SOAR)        ├─Case Mgmt─►: TheHive (Incidents)    :
                                : (10.10.20.20 - Ph. 7) :            : (10.10.20.20 - Ph. 7)  :
                                └ - - - - - - - - - - - ┘            └ - - - - - - - - - - - - ┘
