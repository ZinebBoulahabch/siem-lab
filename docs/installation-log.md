# Installation Log

Technical journal of the lab build, kept phase by phase. One entry per significant action: what was done, how it was checked, and what went wrong. Never write passwords, keys or full configuration backups here.

Conventions: dates in `YYYY-MM-DD`, times in UTC, snapshot names in lowercase (`pfsense-propre`, `dc01-propre`, ...).

---

## Phase 1: Host, virtual networks and pfSense

**Status:** completed. Date: 2026-10-19

### 1.1 Host preparation

| Item | Value |
| --- | --- |
| Host RAM / free SSD | 24 GB / 150 GB free minimum (250 GB recommended) |
| Virtualization (VT-x / AMD-V) | Enabled in the BIOS |
| Hypervisor | VMware Workstation Pro 25.x |
| VM storage folder | `C:\Lab` |

### 1.2 Virtual networks (Virtual Network Editor)

| Network | Type | Subnet | DHCP | Host adapter |
| --- | --- | --- | --- | --- |
| VMnet2 (LAN) | Host-only | 10.10.10.0/24 | Disabled | Not connected |
| VMnet3 (MGMT) | Host-only | 10.10.20.0/24 | Disabled | Connected, IP 10.10.20.254 |
| VMnet4 (DMZ) | Host-only | 10.10.30.0/24 | Disabled | Not connected |
| VMnet5 (ATTACKER) | Host-only | 10.10.99.0/24 | Disabled | Not connected |
| VMnet8 (WAN) | NAT | assigned by VMware | managed by VMware | not used for administration |

### 1.3 Git repository

- Repository `siem-lab` created on GitHub.
- Folder structure: `docs`, `infra`, `detections` (`wazuh`, `sigma`, `yara`), `attacks`, `reports`, `dashboards`.
- `.gitignore` committed before any other file, so no secret can enter the history.

### 1.4 pfSense installation

| Item | Value |
| --- | --- |
| Image | pfSense CE, official ISO from Netgate (checksum verified: yes / no) |
| Resources | 2 GB RAM, 2 vCPU, 20 GB disk, 5 network adapters |

Adapter to interface mapping (check the order in VM settings, it must match):

| VM adapter | VMnet | pfSense interface | Role | pfSense IP |
| --- | --- | --- | --- | --- |
| Network adapter 1 | VMnet8 | em0 | WAN | DHCP |
| Network adapter 2 | VMnet2 | em1 | LAN | 10.10.10.1/24 |
| Network adapter 3 | VMnet3 | em2 | MGMT (OPT1) | 10.10.20.1/24 |
| Network adapter 4 | VMnet4 | em3 | DMZ (OPT2) | 10.10.30.1/24 |
| Network adapter 5 | VMnet5 | em4 | ATTACKER (OPT3) | 10.10.99.1/24 |

### 1.5 Firewall configuration

- Default admin password changed (stored in a password manager, not in this repository).
- Rules created per interface following `infra/network-plan.md`, with logging enabled on block rules.
- The default LAN "allow all" rule was removed and replaced by the rules of the plan.
- Remote syslog set to `10.10.20.10` (UDP 514). The target does not exist yet, which is expected until phase 3.
- Pass rule on MGMT: host `10.10.20.254` to this firewall on TCP 443 (and 22 if SSH is used).

### 1.6 Validation

| Check | Expected | Result |
| --- | --- | --- |
| Host reaches `https://10.10.20.1` | WebGUI login page | OK / KO |
| Temporary VM on LAN reaches the Internet through pfSense | Yes | OK / KO |
| Temporary VM on ATTACKER cannot reach the Internet | No | OK / KO |
| Temporary VM on ATTACKER cannot reach MGMT | No | OK / KO |
| Firewall block events are logged | Entries visible in Status > System Logs > Firewall | OK / KO |

### 1.7 Snapshot

- `pfsense-propre`: taken after the configuration above was validated. Date: `YYYY-MM-DD`.

### 1.8 Issues and fixes

#### Issue: WebGUI unreachable from the host on the MGMT address

- **Symptoms:** ping to `10.10.20.1` worked, but the browser returned `ERR_CONNECTION_REFUSED`, then `ERR_CONNECTION_TIMED_OUT`.
- **Checks that did not explain it:** nginx listening on `*:443`, GUI reset (`resetwebgui`), GUI restart, packet filter disabled (`pfctl -d`).
- **Root cause:** an IP conflict on the Windows host. The `VMware Network Adapter VMnet3` had taken `10.10.20.1`, so the host answered its own pings and refused the connection on port 443.
- **Fix:** set the VMnet3 host adapter to `10.10.20.254/24`, restart the adapter, confirm that VMnet3 is bound to network adapter 3 of the pfSense VM (`em2`).
- **Remaining step:** timeouts after the fix come from the MGMT interface having no pass rule (default deny). Fixed with the MGMT rule of section 1.5.
- **Lesson:** `ping` working does not prove that the right machine answers. Compare the MAC address (`arp -a` on the host, `ifconfig em2` on pfSense) and watch packets with `tcpdump -ni em2`.

### 1.9 Useful commands

```text
# pfSense console, option 8 (shell)
ifconfig em2                       # interface state, IP and MAC
tcpdump -ni em2 host 10.10.20.254  # watch traffic from the admin host
sockstat -l | grep 443             # web server listening
pfctl -d                           # disable the packet filter (test only)
pfctl -e                           # enable the packet filter again
pfctl -s info                      # filter status
```

```text
# Windows host (PowerShell)
ipconfig /all                      # adapter IPs
arp -a | findstr 10.10.20.1        # MAC answering for the pfSense MGMT address
Test-NetConnection 10.10.20.1 -Port 443
```

---

## Phase 2: Active Directory, Windows workstation and DMZ

**Status:** to do.

---

## Entry template for the next phases

**Date:** `YYYY-MM-DD`  **Phase:** `N`  **VM:** `NAME`

- **Action:** what was configured or installed.
- **Command / setting:** the exact command or menu path (no secrets).
- **Check:** how it was verified, with the expected and the observed result.
- **Snapshot:** name of the snapshot taken, if any.
- **Problem and fix:** symptom, root cause, resolution, lesson learned.
