# pfSense build notes

## Why pfSense

pfSense is the firewall and router for the lab. It enforces network segmentation
between corp, mgmt, dmz, and attacker zones, and provides DHCP/DNS/NTP/IDS to
each. All lab traffic between zones and to the internet routes through pfSense.

## Interfaces

| Interface | libvirt network | Role | IP |
|-----------|-----------------|------|-----|
| WAN (vtnet0) | default | Internet egress | DHCP from libvirt |
| LAN (vtnet1) | lab-corp | Corporate users | 192.168.10.1/24 |
| OPT1 (vtnet2) | lab-mgmt | Security team | 192.168.56.1/24 |
| OPT2 (vtnet3) | lab-dmz | Future DMZ | 192.168.20.1/24 |
| OPT3 (vtnet4) | lab-attacker | Attacker | 192.168.100.1/24 |

## Firewall rules summary

- OPT3 (attacker): allow to lab-corp, allow to internet, block to lab-mgmt
- OPT1 (mgmt): allow everywhere
- LAN: default allow LAN to any
- OPT2 (dmz): no rules yet

## Services

- DHCP: enabled on LAN, OPT1, OPT2, OPT3 with /24 pools .100-.200
- NTP: listening on all four lab interfaces, syncing to pool.ntp.org
- Suricata: IDS mode on WAN, ET Open ruleset (malware, exploit, trojan,
shellcode, policy categories)
