# Purple Team Detection Lab

A home lab for purple team detection engineering. Build an Active Directory
environment with intentional misconfigurations, run real attacks against it,
and engineer detections in Wazuh with Sigma rules.

> Status: in progress. Started 28/05/2026. Target completion: 28/06/2026.

## Goals

- 6 VM lab on KVM/QEMU (pfSense, Wazuh, Windows DC, Windows 11, Kali, Ubuntu)
- 20+ MITRE ATT&CK techniques exercised and detected
- 15+ custom Sigma rules
- Full Active Directory attack chain documented as an engagement report
- Three Python automation tools (alert enrichment, coverage reporting, mini SOAR)
- Ansible playbooks that rebuild the entire lab in 30 minutes

## Repository structure

| Directory | What it contains |
|-----------|------------------|
| `docs/` | Architecture, build steps, detection engineering process, incidents |
| `diagrams/` | Network topology, attack chain, detection pipeline diagrams |
| `sigma-rules/` | All custom detection rules in Sigma format |
| `scripts/` | Python automation tooling |
| `ansible/` | Infrastructure as code |
| `reports/` | Engagement report PDF and supporting artifacts |
| `attack-navigator/` | MITRE ATT&CK Navigator coverage layers |

## Status

Updated weekly. See [docs/01-architecture.md](docs/01-architecture.md) for
the current design and [docs/05-mitre-coverage.md](docs/05-mitre-coverage.md)
for the current MITRE coverage map.
