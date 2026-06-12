# Network and infrastructure build notes

These notes record the as-built network layer: the four isolated libvirt networks, the
addressing scheme, and how the host reaches the otherwise-isolated lab segments. The design
rationale lives in `01-network-and-hardware.md`; this file records what was actually defined
and any deviations.

## The four lab networks

All four are fully isolated libvirt networks. Isolated means the network XML has no
`<forward>`, no `<ip>`, and no `<dhcp>` element, so libvirt does no NAT, no routing, and no
DHCP on them. They are pure layer 2 switches. pfSense is the only device that routes between
them and out to the internet, which forces all inter-segment and egress traffic through the
firewall and IDS.

| Network | Bridge | CIDR | Role |
|---|---|---|---|
| `lab-corp` | virbr-corp | 192.168.10.0/24 | Corporate LAN: DC01, WS01, LINTGT01 |
| `lab-mgmt` | virbr-mgmt | 192.168.56.0/24 | Management and monitoring: Wazuh |
| `lab-dmz` | virbr-dmz | 192.168.20.0/24 | DMZ, empty until a later phase |
| `lab-attacker` | virbr-attack | 192.168.100.0/24 | Kali only |

pfSense's WAN side uses libvirt's built-in `default` NAT network (192.168.122.0/24). Only
pfSense touches that network; no lab VM is allowed on it. This is what gives pfSense, and
through it the lab, a path to the internet.

The bridges use `stp="on"` (Spanning Tree Protocol, prevents loops if two NICs are
accidentally attached to one network) and `delay="0"` (skips the STP listening delay so VMs
get connectivity immediately).

The four network XML files are committed under `ansible/inventories/networks/`. Anyone can
recreate the plumbing in one minute by defining and starting them.

## Why isolated and not NAT

A libvirt network with `<forward mode="nat"/>` has libvirt NAT traffic straight to the host's
network, letting a VM reach the internet without passing through pfSense. That would defeat
the point of having pfSense as the firewall and would leave Suricata seeing no traffic. The
original v1 plan made this mistake; the corrected design uses isolated networks with pfSense
as the sole router and egress, which mirrors how a real corporate network is segmented.

## Static IP map

Convention: gateway `.1`, infrastructure `.10` through `.99`, DHCP pool `.100` through
`.200`. pfSense serves DHCP on each lab interface; servers take static or reserved addresses.

| Host | Network | IP | Assignment |
|---|---|---|---|
| pfSense (LAN) | lab-corp | 192.168.10.1 | static gateway |
| pfSense (OPT1) | lab-mgmt | 192.168.56.1 | static gateway |
| pfSense (OPT2) | lab-dmz | 192.168.20.1 | static gateway |
| pfSense (OPT3) | lab-attacker | 192.168.100.1 | static gateway |
| DC01 | lab-corp | 192.168.10.10 | static |
| WS01 | lab-corp | 192.168.10.100 | DHCP reservation |
| LINTGT01 | lab-corp | 192.168.10.20 | static |
| WAZUH01 | lab-mgmt | 192.168.56.10 | static |
| KALI01 | lab-attacker | 192.168.100.50 | DHCP reservation |

## Host access to isolated networks (veth pattern)

Because the lab networks are isolated, the CachyOS host has no route into them. To reach the
pfSense and Wazuh web UIs during the build, the host is patched onto a bridge with a veth
pair (a virtual ethernet cable, two linked interfaces where traffic in one end comes out the
other). This was not spelled out in the original plan and is a deviation worth recording.

Pattern, shown for the mgmt bridge:

```bash
sudo ip link add name virhost-mgmt type veth peer name virmgmt-host
sudo ip link set virmgmt-host master virbr-mgmt   # enslave one end to the bridge
sudo ip link set virhost-mgmt up
sudo ip link set virmgmt-host up
sudo ip addr add 192.168.56.2/24 dev virhost-mgmt  # host IP on the segment
```

The same pattern with `virbr-corp` and `192.168.10.2` gives a leg on corp.

Notes and gotchas:

- These pairs do not survive a host reboot unless a systemd unit recreates them. Re-running
  the commands after a reboot, or on top of an existing pair, throws
  "RTNETLINK answers: File exists." Tear down first with `sudo ip link del virhost-mgmt`
  (deleting one end removes the pair) before recreating, or drive it from a single systemd
  unit instead of by hand.
- Once mgmt firewall rules exist, a single mgmt veth reaches both Wazuh (.56.10) and pfSense
  (.56.1), so the corp veth can be retired. Keeping the host off the corp segment during
  attack runs is also cleaner, since it keeps the host out of packet captures and out of what
  counts as normal corp traffic.

## Host memory tuning

KSM (Kernel Same-page Merging) is enabled on the host. It deduplicates identical memory pages
across VMs, which matters because two Windows VMs share large amounts of identical memory.
This is the difference between scraping by and being comfortable on 32 GB.

```bash
sudo systemctl enable --now ksm
cat /sys/kernel/mm/ksm/pages_sharing   # watch dedup once VMs run
```

## Commit reference

Network XMLs committed in Phase 0 with: `feat(network): define four isolated libvirt networks
for the lab`. Screenshot of `virsh net-list --all` showing all five networks active and
autostart enabled is saved to `diagrams/raw/`.
