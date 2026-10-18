# Network Plan: SIEM & Log Analysis Lab

Reference for the network design of the lab: zones, addressing, virtual machines, data flows and firewall policy. Hypervisor: VMware Workstation Pro. Every network is an isolated virtual network behind a single pfSense firewall.

## 1. Interface & IP Addressing Table

| pfSense interface | Zone | VMware network | Mode | pfSense IP | Subnet | Purpose |
| --- | --- | --- | --- | --- | --- | --- |
| em0 | WAN | VMnet8 | NAT | DHCP (assigned by VMware) | VMware NAT range | Updates and package downloads only |
| em1 | LAN | VMnet2 | Host-only | 10.10.10.1 | 10.10.10.0/24 | Active Directory and workstation |
| em2 | MGMT | VMnet3 | Host-only | 10.10.20.1 | 10.10.20.0/24 | SOC platform (SIEM and tooling) |
| em3 | DMZ | VMnet4 | Host-only | 10.10.30.1 | 10.10.30.0/24 | Exposed vulnerable services |
| em4 | ATTACKER | VMnet5 | Host-only | 10.10.99.1 | 10.10.99.0/24 | Red team machine |

Host PC (administration): VMnet3 virtual adapter, **10.10.20.254/24**, used only to open the Wazuh dashboard (`https://10.10.20.10`).

Isolation settings (VMware Virtual Network Editor):

- DHCP disabled on every VMnet (all machines use fixed IPs).
- "Connect a host virtual adapter" enabled on VMnet3 only; disabled on VMnet2, VMnet4 and VMnet5.
- Only pfSense has a card on VMnet8 (NAT). No other VM is attached to it.

## 2. Virtual Machines Inventory & Fixed IP Assignment

| VM | OS | Zone | Fixed IP | Gateway | DNS | RAM | vCPU | Disk | Role |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| PFSENSE | pfSense CE | all | see section 1 | n/a | n/a | 2 GB | 2 | 20 GB | Firewall, routing, Suricata IDS, syslog client |
| DC01 | Windows Server 2022 (evaluation) | LAN | 10.10.10.10/24 | 10.10.10.1 | 127.0.0.1 (forwarder 10.10.10.1) | 3 GB | 2 | 50 GB | AD DS, DNS, GPO, Sysmon, Wazuh agent |
| WIN10 | Windows 10/11 Enterprise (evaluation) | LAN | 10.10.10.20/24 | 10.10.10.1 | 10.10.10.10 | 3 GB | 2 | 50 GB | Domain workstation, Sysmon, Wazuh agent |
| WAZUH | Ubuntu Server 22.04 | MGMT | 10.10.20.10/24 | 10.10.20.1 | 10.10.20.1 | 6 GB | 4 | 100 GB | Wazuh manager, indexer, dashboard (all-in-one) |
| SOC-TOOLS | Ubuntu Server 22.04 + Docker | MGMT | 10.10.20.20/24 | 10.10.20.1 | 10.10.20.1 | 8 GB | 4 | 80 GB | MISP, Shuffle, TheHive, Cortex (phase 7 only) |
| LNX-DMZ | Ubuntu Server 22.04 | DMZ | 10.10.30.10/24 | 10.10.30.1 | 10.10.30.1 | 2 GB | 2 | 25 GB | Apache, DVWA, Juice Shop, auditd, Wazuh agent |
| KALI | Kali Linux | ATTACKER | 10.10.99.10/24 | 10.10.99.1 | none (isolated) | 3 GB | 2 | 60 GB | Attacker, C2, Atomic Red Team |

Power profiles (host PC with 24 GB RAM, about 5 GB kept for the host):

| Profile | Running VMs | VM RAM |
| --- | --- | --- |
| Build and attack (phases 1 to 6) | PFSENSE, DC01, WIN10, LNX-DMZ, KALI, WAZUH | 19 GB |
| SOC tooling (phase 7) | PFSENSE, DC01, WAZUH, SOC-TOOLS | 19 GB |

Disks are thin-provisioned. Take a clean snapshot of every VM after each successful installation and before each attack series.

## 3. Infrastructure Diagram

```text
                                       ┌───────────────────────┐
                                       │     Internet (WAN)    │
                                       └───────────┬───────────┘
                                                   │ VMnet8 (NAT)
┌──────────────────────────────────────────────────▼─────────────────────────────────────────────────┐
│                                      pfSense Firewall (2 GB)                                       │
│                       Router · Suricata IDS · Syslog client · Zone filtering                       │
│                                   em0 / WAN : DHCP (VMnet8 NAT)                                    │
└────────────┬────────────────────────┬────────────────────────┬────────────────────────┬────────────┘
             │ em1 / LAN              │ em2 / MGMT             │ em3 / DMZ              │ em4 / ATTACKER
             │ VMnet2                 │ VMnet3                 │ VMnet4                 │ VMnet5
             │ 10.10.10.1/24          │ 10.10.20.1/24          │ 10.10.30.1/24          │ 10.10.99.1/24
             ▼                        ▼                        ▼                        ▼
  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
  │ DC01 · 3 GB         │  │ WAZUH · 6 GB        │  │ LNX-DMZ · 2 GB      │  │ KALI · 3 GB         │
  │ 10.10.10.10         │  │ 10.10.20.10         │  │ 10.10.30.10         │  │ 10.10.99.10         │
  │ AD DS / DNS         │  │ Manager / Indexer / │  │ Apache · DVWA ·     │  │ Attacker (Sliver,   │
  │ Sysmon + agent      │  │ Dashboard           │  │ Juice Shop · auditd │  │ Impacket, Hydra)    │
  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘
  ┌─────────────────────┐  ┌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┐
  │ WIN10 · 3 GB        │  ╎ SOC-TOOLS · 8 GB    ╎
  │ 10.10.10.20         │  ╎ 10.10.20.20         ╎
  │ Workstation         │  ╎ MISP · Shuffle ·    ╎
  │ Sysmon + agent      │  ╎ TheHive (phase 7)   ╎
  └─────────────────────┘  └╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┘

  Host PC (administration): VMnet3 adapter 10.10.20.254, attached to the MGMT network
  (reaches WAZUH and SOC-TOOLS). Dashed box: component started in phase 7 only.
```

## 4. Logical Data & Detection Flow

```text
  Sources                      Collection                    Analysis                      Response
  ───────                      ──────────                    ────────                      ────────
  Windows (DC01, WIN10)  ──┐
    Sysmon, 4688, 4104     │   Wazuh agents
                           ├─► 1514/TCP (events)  ──┐
  Linux (LNX-DMZ)          │   1515/TCP (enrolment) │
    auditd, Apache, FIM  ──┘                        │      ┌───────────────────────┐
                                                    ├────► │ Wazuh manager         │
  pfSense                                           │      │ decode · rules ·      │
    Suricata EVE alerts ───► syslog 514/UDP  ───────┘      │ ATT&CK mapping        │
    firewall logs                                          └───────┬───────┬───────┘
                                                                   │       │
                                   MISP (IoC lists, CDB) ──────────┘       │
                                                                           ▼
                                                                  Wazuh indexer
                                                                  (dashboards, MITRE view)
                                                                           │
                                                          alerts level 10+ │
                                                                           ▼
                                                                  Shuffle (SOAR)
                                                                  │      │      │
                                                                  ▼      ▼      ▼
                                                         Cortex /   TheHive   Active Response
                                                         AbuseIPDB   (case)   (Wazuh) or
                                                         (enrich IP)          pfSense API (block)
```

Flow summary:

1. **Collect**: Wazuh agents (Windows, Linux) send events to the manager; pfSense sends syslog (firewall and Suricata alerts).
2. **Analyze**: the manager decodes events, applies custom rules (Wazuh XML and converted Sigma) and tags each alert with its MITRE ATT&CK technique.
3. **Enrich**: IoCs published in MISP are exported as CDB lists and matched against incoming events.
4. **Index and visualize**: alerts are stored in the indexer and shown in the dashboards.
5. **Respond** (phase 7): alerts of level 10 and above go to Shuffle, which enriches the source IP, opens a TheHive case and triggers a block (Wazuh Active Response or pfSense API).

| Flow | Source | Destination | Port / protocol |
| --- | --- | --- | --- |
| Agent enrolment | DC01, WIN10, LNX-DMZ | WAZUH | 1515/TCP |
| Agent events | DC01, WIN10, LNX-DMZ | WAZUH | 1514/TCP |
| Firewall and Suricata logs | pfSense | WAZUH | 514/UDP |
| Dashboard access | Host PC (10.10.20.254) | WAZUH | 443/TCP |
| Alert to playbook | WAZUH | SOC-TOOLS (Shuffle webhook) | HTTPS (port set in Shuffle) |
| IP block via API | SOC-TOOLS | pfSense | 443/TCP |

## 5. Firewall Filtering Policy Guidelines

Principles:

- Default deny on every interface. pfSense allows all traffic on LAN by default; **delete that default rule** and replace it with the rules below.
- Rules are evaluated top to bottom: place block rules first.
- Enable logging on every block rule so denied traffic reaches Wazuh through syslog.
- The ATTACKER zone never reaches the Internet or the MGMT zone.
- Scenario rules marked "disabled by default" are switched on only for the matching attack test, then switched off again.

**ATTACKER interface (em4)**

| Action | Source | Destination | Port | State |
| --- | --- | --- | --- | --- |
| Block (log) | 10.10.99.0/24 | MGMT net, WAN | any | enabled |
| Pass | 10.10.99.0/24 | DMZ net | any | enabled |
| Pass | 10.10.99.0/24 | LAN net | any | disabled by default (assumed-compromise scenarios) |
| Block (log) | any | any | any | enabled (implicit default) |

**DMZ interface (em3)**

| Action | Source | Destination | Port | State |
| --- | --- | --- | --- | --- |
| Pass | LNX-DMZ | WAZUH | 1514/TCP, 1515/TCP | enabled |
| Pass | DMZ net | pfSense (DNS) | 53/UDP, 53/TCP | enabled |
| Pass | DMZ net | WAN | 80/TCP, 443/TCP | enabled during installation, then restrict to updates |
| Pass | DMZ net | LAN net | 445/TCP, 3389/TCP | disabled by default (lateral movement scenario) |
| Block (log) | any | any | any | enabled (implicit default) |

**LAN interface (em1)**

| Action | Source | Destination | Port | State |
| --- | --- | --- | --- | --- |
| Pass | DC01, WIN10 | WAZUH | 1514/TCP, 1515/TCP | enabled |
| Pass | LAN net | pfSense (DNS) | 53/UDP, 53/TCP | enabled |
| Pass | LAN net | WAN | 80/TCP, 443/TCP | enabled during installation, then restrict to updates |
| Block (log) | LAN net | ATTACKER net | any | enabled |
| Block (log) | any | any | any | enabled (implicit default) |

**MGMT interface (em2)**

| Action | Source | Destination | Port | State |
| --- | --- | --- | --- | --- |
| Pass | MGMT net | WAN | 80/TCP, 443/TCP | enabled (updates, container images) |
| Pass | MGMT net | pfSense (DNS) | 53/UDP, 53/TCP | enabled |
| Pass | SOC-TOOLS | pfSense | 443/TCP | enabled in phase 7 (API block action) |
| Block (log) | MGMT net | ATTACKER net | any | enabled |
| Block (log) | any | any | any | enabled (implicit default) |

**Router-originated and host traffic**

| Action | Source | Destination | Port | Note |
| --- | --- | --- | --- | --- |
| Allow | pfSense | WAZUH | 514/UDP | Syslog remote logging, set in Status > System Logs > Settings |
| Allow | Host PC (10.10.20.254) | WAZUH, SOC-TOOLS | 443/TCP and tool ports | Administration from the host only |

**Hardening checklist**

- Change the default pfSense admin password and store it in a password manager (never in Git).
- Restrict the pfSense web interface to the MGMT interface.
- Enable Suricata on the LAN and DMZ interfaces with the ET Open ruleset, and send its alerts to syslog.
- After installation is complete, remove the broad WAN rules and keep only what updates need.
- Take a pfSense snapshot after every rule change that works (`pfsense-clean`, `pfsense-rules-v2`, ...).
- Check isolation before any attack test: a temporary VM on the ATTACKER network must not reach the Internet, MGMT or the host network.
