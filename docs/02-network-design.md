# Network design

## Goals

The lab network design has three goals:

1. Isolate the lab from your host's real network and from the internet, except through one controlled chokepoint (pfSense).
2. Enforce realistic segmentation between corporate, management, DMZ, and attacker zones.
3. Generate inspectable network telemetry on every cross-zone and outbound connection.

## Segments

| Segment | Subnet | Role | Hosts |
|---------|--------|------|-------|
| lab-corp | 192.168.10.0/24 | Corporate users | DC01 (10.10), WS01 (10.100), LINTGT01 (10.20) |
| lab-mgmt | 192.168.56.0/24 | Security tooling | WAZUH01 (56.10) |
| lab-dmz | 192.168.20.0/24 | DMZ (future, internet-facing) | none yet |
| lab-attacker | 192.168.100.0/24 | Attacker position | KALI01 (100.50) |

pfSense gateways: corp .10.1, mgmt .56.1, dmz .20.1, attacker .100.1.

## Why isolated libvirt networks

The lab segments are defined as **isolated** libvirt networks (no `<forward>` element, no libvirt-assigned IPs, no libvirt DHCP). This is a deliberate choice over libvirt's NAT mode.

A NAT-mode libvirt network would route VMs straight to the internet through the host, bypassing pfSense, and libvirt's built-in DHCP would compete with pfSense for control of each subnet. The lab would look firewalled but actually wouldn't be: traffic would route around the firewall and Suricata would see almost nothing.

With isolated networks, pfSense is the only path between segments and the only path out. The firewall is genuinely the chokepoint it appears to be.

## Firewall policy matrix

| Source | Destination | Policy |
|--------|-------------|--------|
| lab-attacker | lab-corp | Allow (assumed-breach scenario: attacker has internal foothold) |
| lab-attacker | lab-mgmt | Block (attacker should not see the SIEM, modeling real attacks) |
| lab-attacker | internet | Allow (attackers fetch tools) |
| lab-corp | lab-mgmt | Allow only Wazuh agent ports (TCP 1514, 1515, 55000) |
| lab-corp | internet | Allow |
| lab-mgmt | everywhere | Allow (admin position) |
| All outbound | internet | Inspected by Suricata in IDS mode |

## Assumed-breach scenario

The attacker (Kali on lab-attacker) is presumed to already have internal network access. Kerberoasting, AS-REP roasting, and unauthenticated AD reconnaissance all require the attacker to reach the DC's Kerberos, LDAP, and SMB ports, which a true external attacker cannot do without first gaining a foothold. Assumed-breach is the standard framing for internal pentests and the most common starting point for real ransomware intrusions, so the lab models that explicitly.

## How the host reaches the lab

Because the lab segments are isolated, the CachyOS host has no built-in route to them. Two options are supported:

1. **veth pair** (current default): create a virtual ethernet pair giving the host a static IP on the management bridge (192.168.56.2). Documented in Phase 0. Must be re-run after each host reboot or made into a one-shot systemd unit.
2. **pfSense as jump host**: SSH into pfSense and use its `pfctl` or SSH local forwarding to reach the lab. Cleaner but requires SSH access to pfSense, which is configured in Phase 1.

The veth approach is documented in the build notes as the day-to-day path.

![Network topology](../diagrams/network-topology.png)
