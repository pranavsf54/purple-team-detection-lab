# Wazuh build notes

Repo path: `docs/03-build/05-wazuh.md`

## Role in the lab

Wazuh is the SIEM (Security Information and Event Management platform) for the lab. It is
the single place where telemetry from every monitored host is collected, run against
detection rules, and turned into alerts. It is also the host intrusion detection system,
the file integrity monitor, and the vulnerability scanner for the environment, all on one
platform. Everything in Phases 2 through 6 reports into this box.

The Wazuh platform is three services installed together by the all-in-one installer:

- **Wazuh Manager (server).** Receives events from agents, runs detection rules, raises
  alerts, exposes the REST API.
- **Wazuh Indexer.** The OpenSearch-based search engine that stores events for searching and
  dashboards.
- **Wazuh Dashboard.** The web UI analysts work in.

In production these are split across hosts and clustered. For the lab they share one VM,
which is fine at this scale.

## Version and OS

- Wazuh 4.14.5, all-in-one install.
- Ubuntu Server 22.04 LTS.

### Why this OS (decision record)

The build first targeted Ubuntu 26.04, then settled on 22.04. The reasoning is worth keeping
because it is a real engineering judgement, not an accident:

- Ubuntu 26.04 (April 2026 LTS) is not on Wazuh's supported operating system list. There is an
  open Wazuh issue to add it, targeted at a future point release and currently blocked.
- The SIEM is the one box the entire lab depends on for weeks, so it is the wrong place to run
  an OS the vendor has not finished testing.
- 22.04 is on the supported list, is well understood, and uses the simpler single-file chrony
  layout (no `sources.d` drop-in), which avoids the time-sync path issue that the newer
  releases introduce.

## Resources

| Setting | Value |
|---|---|
| RAM | 8 GB (verify against your actual `virt-install`; the original plan said 6 GB, which causes OpenSearch garbage-collection stalls under verbose Sysmon load) |
| vCPU | 2 |
| Disk | 80 GB qcow2, thin provisioned |
| Networks | one NIC on `lab-mgmt` only |

Wazuh sits on exactly one network and reaches anything else through pfSense, unlike pfSense
which has a foot in every segment.

## Storage and LVM

Installed with the default LVM layout. Ubuntu's installer leaves part of the volume group
unallocated, so the root logical volume was extended to the full disk after first boot:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Disk encryption was deliberately not used, to avoid a passphrase prompt on every boot in a
lab where VMs start and stop constantly.

Forward note: the Wazuh indexer grows by gigabytes per week under Sysmon load. An Index State
Management (ISM) retention policy to delete indices older than 14 to 30 days is set in Phase
2. Without it this disk fills silently.

## Network

| Setting | Value |
|---|---|
| Address | 192.168.56.10/24 (intended static; confirm this is a static netplan address and not a DHCP lease from the OPT1 pool) |
| Gateway | 192.168.56.1 (pfSense lab-mgmt interface) |
| DNS | 1.1.1.1, 9.9.9.9 (temporary external resolvers; revisit once DC01 provides internal DNS, because domain name resolution will need to point at the DC) |

## Components and ports

| Component | Port | Exposure |
|---|---|---|
| Wazuh Manager, agent connections | 1514 | reachable by agents |
| Wazuh Manager, agent enrollment | 1515 | reachable by agents |
| Wazuh Manager, REST API | 55000 | management |
| Wazuh Indexer (OpenSearch) | 9200 | internal only |
| Wazuh Dashboard | 443 | HTTPS web UI |

## Time synchronization

chrony installed and configured to use pfSense (192.168.56.1) as its NTP source, replacing
the default Ubuntu pool servers.

Sync state: **pending verification.** During the build, pfSense was returning zero NTP replies
(`chronyc ntpdata 192.168.56.1` showed Total RX 0), which pointed at the pfSense side rather
than the client. Confirm with `chronyc sources -v` that the `192.168.56.1` row shows `^*`
before relying on this box for anything time-sensitive. This must be working before DC01 is
promoted, because Active Directory uses Kerberos and Kerberos rejects tickets when clocks
drift more than five minutes apart. See the troubleshooting playbook for the diagnosis path.

## Reaching the dashboard from the host

The lab-mgmt network is isolated, so the CachyOS host has no route to 192.168.56.10 by
default. Access is via a veth pair that patches the host onto the mgmt bridge:

```bash
sudo ip link add name virhost-mgmt type veth peer name virmgmt-host
sudo ip link set virmgmt-host master virbr-mgmt
sudo ip link set virhost-mgmt up
sudo ip link set virmgmt-host up
sudo ip addr add 192.168.56.2/24 dev virhost-mgmt
```

This is not persistent across a host reboot unless a systemd unit recreates it. If a unit is
in place, drive it with `systemctl restart lab-veth.service` rather than running the commands
by hand, to avoid the "RTNETLINK answers: File exists" error from stacking duplicate pairs.

## Initial state

- No agents enrolled. Agent enrollment is Phase 2 work.
- Default Wazuh ruleset loaded; built-in MITRE ATT&CK mapping present but mostly empty until
  agents report.
- No custom Sigma or local rules yet.

## Issues hit during this build

Cross-referenced in `docs/13-troubleshooting-playbook.md`:

- chrony source path differs on newer Ubuntu (`/etc/chrony/sources.d/` rather than
  `chrony.conf`); not relevant on the final 22.04 build but cost time on the 24.04 attempt.
- NTP returning Total RX 0 from pfSense; pfSense-side, not client-side.
- Dashboard unreachable from host; root cause was the VM not holding the expected
  192.168.56.10 address and stale veth state on the host.

## Snapshots

| Name | State captured |
|---|---|
| `base-install` | Clean Ubuntu 22.04 with updates and chrony pointed at pfSense |
| `wazuh-installed` | Wazuh 4.14 installed, admin login confirmed, no agents yet |

## Screenshots captured

- Wazuh dashboard first view, saved to `diagrams/raw/`. A keeper for the README.
