# Purple Team Active Directory Detection Lab

![Network topology](diagrams/network-topology.png)


A fully isolated, network-segmented Active Directory lab for practicing the complete
**attack -> detect -> tune** cycle against a monitored Windows domain. Every host is built on
KVM/libvirt, segmented behind a pfSense firewall with Suricata IDS, and monitored by a Wazuh
SIEM. Misconfigurations are seeded deliberately, attacked from a Kali box, and then turned into
detections — so each technique is demonstrated end to end, from adversary action to the alert
that catches it.

## What this project demonstrates

- Virtualization and infrastructure: KVM/QEMU/libvirt, isolated networks, host tuning (KSM),
  snapshot-driven workflows.
- Network engineering: four-zone segmentation with a single routing/IDS chokepoint.
- Detection engineering: SIEM deployment, log pipelines, MITRE ATT&CK-mapped detections.
- Offensive security: Active Directory attack chains (Kerberos abuse, credential theft,
  ACL abuse) executed and understood, not just run.
- Infrastructure as code and documentation discipline: reproducible builds, decision records.

## Architecture

pfSense is the only device that routes between zones and to the internet, which forces all
inter-segment and egress traffic through the firewall and Suricata.

| Network | CIDR | Role | Hosts |
|---|---|---|---|
| `lab-corp` | 192.168.10.0/24 | Corporate LAN | DC01, WS01, LINTGT01 |
| `lab-mgmt` | 192.168.56.0/24 | Monitoring plane | WAZUH01 |
| `lab-dmz` | 192.168.20.0/24 | DMZ (future phase) | — |
| `lab-attacker` | 192.168.100.0/24 | Adversary | KALI01 |

pfSense holds a gateway interface on every zone (`.1`) plus a WAN leg on libvirt's NAT network
for egress. Full addressing is in `docs/03-build/00-networks.md`.

## Components

| Host | Role | Platform |
|---|---|---|
| pfSense | Firewall, router, DHCP/DNS/NTP, Suricata IDS | pfSense CE |
| WAZUH01 | SIEM, HIDS, FIM, vuln scanner | Wazuh 4.14.5 on Ubuntu 22.04 LTS |
| DC01 | Domain controller, `corp.lab.lan` | Windows Server 2022 (2016 functional level) |
| KALI01 | Offensive launch platform | Kali Linux 2026.1 |
| WS01 | Domain workstation | Windows 11 (domain-joined) |
| LINTGT01 | Linux domain target | Ubuntu 22.04 (domain-joined, DVWA) |

## Methodology

The lab runs a purple-team loop, one misconfiguration at a time:

1. **Seed** a realistic weakness into the domain.
2. **Attack** it from KALI01 as a low-privilege user, generating real adversary telemetry.
3. **Detect** by writing or tuning a rule (Wazuh/Sigma) that catches that telemetry.
4. **Tune** to remove false positives against the BadBlood-generated background noise.

BadBlood populates the domain with roughly 2,500 users, hundreds of groups, and randomized
ACLs, so detections must find the planted needle without lighting up on benign accounts.

## Seeded attack surface (Phase 4 chain)

These weaknesses are intentional and exist only inside the isolated lab. The credentials are
deliberately weak and must never be reused anywhere.

| # | Misconfiguration | Account / object | MITRE ATT&CK |
|---|---|---|---|
| 1 | Kerberoastable service account | `sqlsvc` (SPN set) | T1558.003 |
| 2 | AS-REP roastable account | `jsmith` (no pre-auth) | T1558.004 |
| 3 | GPP cleartext password in SYSVOL | `corp\backup-svc` | T1552.006 |
| 4 | Weak ACL (GenericWrite) | HOLLIE_MANN -> BUDDY_MCCARTY | T1098 |

## Phase roadmap

| Phase | Scope | Status |
|---|---|---|
| 0 | Isolated libvirt networks, repo, diagram | Complete |
| 1 | Infrastructure build (pfSense, Wazuh, DC01, WS01, Kali, LINTGT01) + seeded misconfigurations | Complete |
| 2 | Wazuh agents, Sysmon via GPO, PowerShell logging, audit policy, baseline, first detection dry-run, CI harness | In progress |
| 3 | Atomic Red Team campaign, 15+ techniques detected and tuned | Planned |
| 4 | Full Active Directory attack chain (AS-REP, Kerberoast, ACL abuse, DCSync, Golden Ticket) | Planned |
| 5 | Threat hunts, staged incidents, NIST 800-61 reports, Volatility 3 memory capture | Planned |
| 5.5 | Cloud thin slice: Microsoft Sentinel trial, KQL detections, Entra ID identity detections | Planned |
| 6 | Python automation (enrichment, coverage reporter, mini-SOAR + AI stage), Ansible | Planned |
| 7 | README polish, PDF engagement report, walkthrough video, blog posts | Planned |
| 8 | Full cloud phase, full identity coverage | Planned |

## Repository layout

```
docs/
  03-build/
    00-networks.md
    01-pfsense.md
    02-wazuh.md
    03-domain-controller.md
    04-workstation-ws01.md
    05-attacker-kali.md
    06-linux-target-lintgt01.md
  13-troubleshooting-playbook.md
ansible/
  inventories/networks/
    lab-attacker.xml
    lab-corp.xml
    lab-dmz.xml
    lab-mgmt.xml
  playbooks/
  roles/
diagrams/
  raw/
sigma-rules/
  linux/
  network/
  windows/
tests/
  README.md
  sample-events/
scripts/
  coverage-reporter/
  enrichment/
  mini-soar/
reports/
README.md
LICENSE
```

## Rebuilding the lab

1. Define and start the four isolated networks from `ansible/inventories/networks/`.
2. Deploy pfSense, attach a leg to each zone, apply the rules in `01-pfsense.md`.
3. Build WAZUH01, DC01, WS01, KALI01, and LINTGT01 per their build notes.
4. Patch the host onto an isolated segment with a veth pair to reach the web UIs
   (see `00-networks.md`).

## Safety and scope

This lab is fully isolated: no lab VM touches the internet except through pfSense, and the
attacker zone cannot reach the monitoring plane. All credentials in this repository are
intentionally weak lab values for seeded misconfigurations. No real secrets — host passwords,
the Wazuh admin password, API keys, or Ansible vault material — are committed; see
`.gitignore`.

## Status

Phase 1 complete: all six VMs built, the `corp.lab.lan` domain promoted, BadBlood run, and
the four misconfigurations seeded and documented. Phase 2 is in progress: Wazuh agents are
enrolled and reporting, Sysmon is deployed via GPO with the sysmon-modular config and its
events are reaching Wazuh, PowerShell Script Block and Module Logging are enabled, and the
"Audit Directory Service Access" policy plus the domain-root SACL are in place for the Phase 4
DCSync detection. A 48-hour baseline period is running to capture the normal noise floor
before any attacks. Remaining Phase 2 items: close out the baseline noise-floor write-up,
develop the first detection end to end as a dry run (Kerberoasting), and stand up the CI
rule-testing harness.

## Tech stack

<!-- shields.io badges. Generate at https://shields.io. Example below. -->
![KVM](https://img.shields.io/badge/Hypervisor-KVM%2FQEMU-red)
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-blue)
![Sigma](https://img.shields.io/badge/Detections-Sigma-green)
![Python](https://img.shields.io/badge/Automation-Python-yellow)
![Sentinel](https://img.shields.io/badge/Cloud-Microsoft%20Sentinel-blueviolet)

## About

Built by Pranav Fangoo, https://ca.linkedin.com/in/pranav-fangoo. 

--- 
> *“Hard work beats talent when talent doesn't work hard.”* - **Tim Notke**