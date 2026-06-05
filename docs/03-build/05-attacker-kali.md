# Attacker box (KALI01) build notes

Repo path: `docs/03-build/04-attacker-kali.md`

These notes record the as-built offensive host: its placement on the isolated attacker
segment, why the firewall lets it reach corp but not mgmt, and the toolset installed for the
Phase 4 attack chain. The attack methodology itself lives in the Phase 4 notes; this file
records what the box *is*.

## Role in the lab

KALI01 is the single offensive host. It is the only device on `lab-attacker`
(192.168.100.0/24) and is the box every Phase 4 attack is launched from, playing the part of
an adversary who has gained a foothold in the environment. It does not host services; it is a
launch platform for enumeration, credential attacks, and post-exploitation tooling.

Crucially, Kali starts each attack run as the equivalent of a low-privilege domain user (using
one of the seeded accounts). It is never given domain-admin shortcuts. The whole point of the
lab is to walk the path from "nobody on the network" to "domain admin" and watch what the SIEM
sees at each step.

## Segmentation (decision record)

The pfSense OPT3 (attacker) rules are: allow to `lab-corp`, allow to internet, **block to
`lab-mgmt`**. That mgmt block is deliberate and worth keeping:

- A real attacker should not be able to reach the security team's monitoring plane. Letting
  Kali touch Wazuh (192.168.56.10) would be an unrealistic shortcut and, worse, would let
  attacker traffic contaminate the very telemetry the lab is trying to evaluate.
- It also protects the integrity of detections. If Kali could reach the SIEM, a "did the alert
  fire?" result could no longer be trusted, because the attacker host would be inside the
  monitored boundary in a way no real one would be.

So traffic flows Kali -> pfSense -> corp (monitored), and Kali -> pfSense -> internet, but
never Kali -> mgmt. This mirrors the segmentation in `01-pfsense.md`.

## Version and OS

- Kali Linux 2026.1 (rolling release).
- Architecture: amd64 (Ryzen 5 5600X host).

## Resources

| Setting | Value |
|---|---|
| RAM | 4 GB (verify against your actual `virt-install`; raise to 6 GB only if BloodHound CE's backend is run on-box and memory allows under the 32 GB host budget) |
| vCPU | 2 |
| Disk | 40 GB qcow2, thin provisioned |
| Networks | one NIC on `lab-attacker` only |

Like Wazuh, Kali sits on exactly one network and reaches everything else through pfSense.

## Network

| Setting | Value |
|---|---|
| Address | 192.168.100.50 (DHCP reservation from the OPT3 pool; confirm the reservation is pinned on pfSense and not a floating lease) |
| Gateway | 192.168.100.1 (pfSense lab-attacker interface) |
| DNS | 192.168.100.1 (pfSense) for general resolution |

Note on AD name resolution: to resolve `corp.lab.lan` and locate DC01, point DNS at the
domain controller (192.168.10.10) during attack runs, or pass the DC explicitly to each tool
(`-ns 192.168.10.10` / `--dc-ip`). Most AD tooling here takes the DC IP as a flag, so changing
the system resolver is usually unnecessary, but Kerberos and SPN-based attacks behave more
predictably when the box can resolve the domain FQDN.

## Offensive toolset

Installed from the Kali repo in one shot, plus one binary that is not packaged:

```bash
sudo apt update
sudo apt install -y bloodhound bloodhound-ce-python impacket-scripts \
  netexec hashcat responder evil-winrm smbclient
```

kerbrute is not in the Kali apt repo and is installed as a binary from the official releases:

```bash
curl -L -o kerbrute https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute
sudo mv kerbrute /usr/local/bin/
kerbrute version
```

## BloodHound CE notes

- `bloodhound` (apt) is the CE analysis app — the graph UI used to find attack paths.
- `bloodhound-ce-python` is the collector run from Kali against DC01 as the foothold user; it
  produces the JSON/zip ingested into the app.
- BloodHound CE runs on its own backend (Postgres + a graph database) rather than the legacy
  single neo4j the old version used. Confirm the backend comes up on first launch before
  relying on it; record the exact start procedure here once verified.

## Wordlists

`rockyou` is the primary cracking list for the Kerberoast (`sqlsvc` / `Summer2024!`) and
AS-REP (`jsmith` / `Welcome123!`) hashes, both of which are present in it.

## Snapshots

| Name | State captured |
|---|---|
| `base-install` | Clean Kali 2026.1 with updates, on lab-attacker, reservation confirmed |
| `tools-installed` | Full offensive toolset + kerbrute binary present and version-checked |

## Issues hit during this build

Cross-referenced in `docs/13-troubleshooting-playbook.md`:

- `bloodhound-ce` and `kerbrute` apt install failures ("Unable to locate package"); resolved by
  the corrected package name and the GitHub binary respectively.

## Screenshots captured

- `kerbrute version` output and BloodHound CE first launch, saved to `diagrams/raw/`.

## Commit reference

Attacker box documented with: `feat(attacker): build KALI01 offensive host and document
toolset for the Phase 4 attack chain`.
