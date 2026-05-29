# Network design

## Goals

The lab network is designed to (1) isolate the lab from the host's real network
and the internet except through one controlled chokepoint, (2) enforce realistic
segmentation between corporate, management, DMZ, and attacker zones, and (3)
generate network-layer telemetry on every cross-zone connection.

## Segments

| Segment | Subnet | Purpose | Hosts |
|---------|--------|---------|-------|
| lab-corp | 192.168.10.0/24 | Corporate network | DC01, WS01, LINTGT01 |
| lab-mgmt | 192.168.56.0/24 | Security tooling | WAZUH01 |
| lab-dmz | 192.168.20.0/24 | Internet-facing (future) | none yet |
| lab-attacker | 192.168.100.0/24 | Attacker position | KALI01 |

## Why isolated libvirt networks

Each lab network is an isolated libvirt network with no NAT of its own. This
forces all routing through pfSense, which means the firewall actually sees and
can filter every cross-zone and outbound connection. A common mistake is to
NAT each network directly through the host, which bypasses the firewall
entirely and produces no inspectable egress traffic.

## Firewall policy

- Attacker can reach corp (to attack) and the internet (to fetch tools).
- Attacker CANNOT reach management. A real intruder would not have visibility
  into the SIEM, and this models that.
- Management can reach everything (the admin position).
- All egress passes Suricata in IDS mode on the WAN interface.

![Network topology](../diagrams/network-topology.png)
