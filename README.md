# SIEM & Log Analysis Lab

Open-source SOC lab built on a single workstation: a segmented 7-VM environment monitored by **Wazuh** (SIEM/XDR), with network detection (**Suricata**), threat intelligence (**MISP**), case management (**TheHive**) and automated response (**Shuffle**). Detections are written as code, mapped to **MITRE ATT&CK**, and validated by simulated attacks (purple teaming).

> Status: work in progress. Phase 1 (host, virtual networks, firewall) is complete.

## Goals

- Build a realistic monitoring environment: Windows (Active Directory), Linux, firewall and network telemetry feeding one SIEM.
- Write and version at least 20 detections mapped to MITRE ATT&CK techniques.
- Validate each detection with a real attack, measure coverage, then fix the gaps (detect, test, improve).
- Extract indicators of compromise (IoCs) from the simulated attacks and share them in MISP (STIX 2.1).
- Automate part of the response (enrichment, case creation, IP blocking) with a SOAR playbook.
- Produce an incident report and runbooks written like a SOC analyst would.

## Architecture overview

Everything runs in VMware Workstation. **pfSense** is the only link between four isolated zones, and the attacker network has no access to the Internet or to the SOC management network.

| Zone | Network | Machines |
| --- | --- | --- |
| LAN | 10.10.10.0/24 | DC01 (Active Directory, DNS), WIN10 (domain workstation) |
| MGMT | 10.10.20.0/24 | WAZUH (SIEM), SOC-TOOLS (MISP, Shuffle, TheHive, Cortex) |
| DMZ | 10.10.30.0/24 | LNX-DMZ (Apache, DVWA, Juice Shop) |
| ATTACKER | 10.10.99.0/24 | KALI (attack tools, C2) |

```text
                 Internet (VMnet8 NAT)
                          │
                 ┌────────▼────────┐
                 │ pfSense + IDS   │
                 └─┬──────┬─────┬──┬┘
          LAN ─────┘  MGMT│  DMZ│  └──── ATTACKER
     DC01 · WIN10   WAZUH · │ LNX-DMZ       KALI
                    SOC-TOOLS
```

Data flow: agents (Windows, Linux) and syslog (pfSense, Suricata) send events to Wazuh, which decodes them, applies detection rules tagged with ATT&CK techniques, and forwards high-severity alerts to Shuffle for enrichment, ticketing and blocking.

The full addressing table, firewall policy and detailed diagrams are in [`infra/network-plan.md`](infra/network-plan.md).

## Repository structure

```text
siem-lab/
├── README.md
├── .gitignore
├── docs/          Network plan notes, intentional vulnerabilities, log source coverage, installation log
├── infra/         Network plan, VM inventory, build and configuration notes
├── detections/
│   ├── wazuh/     Custom Wazuh rules (XML)
│   ├── sigma/     Sigma rules
│   └── yara/      YARA rules for test payloads
├── attacks/       One file per simulated technique: command, logs, alert, fix
├── reports/       Incident report, ATT&CK Navigator layers (before and after)
└── dashboards/    Dashboard captures and exports
```

## Hardware requirements

| Resource | Minimum | Notes |
| --- | --- | --- |
| RAM | 24 GB | About 19 GB for the running VMs, the rest for the host |
| Storage | 300 GB free SSD | VM disks are thin-provisioned (about 385 GB allocated at most) |
| CPU | 4 cores or more | Virtualization (VT-x / AMD-V) enabled in the BIOS |
| Hypervisor | VMware Workstation Pro | Free for personal use |

| Virtual machine | RAM |
| --- | --- |
| pfSense | 2 GB |
| DC01 (Windows Server 2022, evaluation) | 3 GB |
| WIN10 (Windows 10/11, evaluation) | 3 GB |
| LNX-DMZ (Ubuntu Server 22.04) | 2 GB |
| KALI | 3 GB |
| WAZUH (Ubuntu Server 22.04) | 6 GB |
| SOC-TOOLS (Ubuntu Server 22.04 + Docker, phase 7 only) | 8 GB |

Two power profiles keep the host within 24 GB: phases 1 to 6 run every VM except SOC-TOOLS; phase 7 runs pfSense, DC01, WAZUH and SOC-TOOLS.

## Roadmap

| Phase | Content | Status |
| --- | --- | --- |
| 1 | Host, virtual networks, Git repository, pfSense | Done |
| 2 | Active Directory, Windows workstation, DMZ server | Planned |
| 3 | Wazuh deployment and agents | Planned |
| 4 | Telemetry (Sysmon, auditd, Suricata, firewall logs) | Planned |
| 5 | Custom Wazuh and Sigma detections mapped to ATT&CK | Planned |
| 6 | Attack simulation and purple teaming | Planned |
| 7 | MISP, TheHive, Shuffle, IoC extraction and automated response | Planned |
| 8 | Dashboards, investigation and incident report | Planned |

## Tools

Wazuh · Suricata · pfSense · Sysmon · auditd · Sigma · YARA · MISP · TheHive · Cortex · Shuffle · Atomic Red Team · MITRE ATT&CK Navigator · Kali Linux

## Safety and ethics

- All attacks target only the machines of this lab, inside isolated virtual networks.
- The attacker network has no Internet access and no access to the host or to the management network.
- Windows images are the official Microsoft evaluation versions.
- No password, key, certificate or sensitive export is committed to this repository (see `.gitignore`).
- The vulnerable applications (DVWA, Juice Shop) and the weak accounts exist on purpose and are documented in `docs/`.

## License

[MIT](LICENSE)
