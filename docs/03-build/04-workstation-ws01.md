# Workstation (WS01) build notes

As-built record of the Windows 11 client: the TPM/UEFI emulation needed to install it, its
domain join, and the user accounts on it. The step-by-step lives in
`docs/04-phase-1-infrastructure.md` Section 4; this file is the distilled record.

## Role in the lab

WS01 is the "employee laptop" — a domain-joined Windows 11 client that represents a normal
corporate endpoint. It is one of the primary attack targets and the on-domain vantage point
for tooling that has to run from inside the domain (BloodHound enumeration, lateral movement,
post-exploitation). Under the lab's assumed-breach scoping, WS01 plus the `alice.miller`
identity is the realistic foothold the attacker operates from.

It is Windows 11 specifically (not 10) because that is what modern corporate fleets look like
in 2026, with tighter security defaults and the mandatory TPM 2.0 requirement.

## Version and OS

- Windows 11 (Enterprise/Pro evaluation ISO, English International).
- `--os-variant win11`.

## Resources

| Setting | Value |
|---|---|
| RAM | 4 GB (4096 MB) |
| vCPU | 2 |
| Disk | 80 GB qcow2, thin provisioned, virtio bus |
| Networks | one NIC on `lab-corp` |
| Firmware | UEFI, with `smm.state=on` |
| TPM | emulated 2.0 via swtpm (`backend.type=emulator,backend.version=2.0,model=tpm-crb`) |

### Windows 11 install requirements (decision record)

Two flags exist only because Windows 11 refuses to install without them, and both cost time if
omitted:

- **Emulated TPM 2.0.** Windows 11 will not install without a TPM 2.0 present. The VMs have no
  real chip, so swtpm emulates one. `smm.state=on` (System Management Mode) is also enabled
  because the Windows 11 secure-boot path expects it. `kvm_hidden=on` hides the hypervisor from
  the guest.
- **VirtIO driver ISO attached as a second CD-ROM.** Windows does not ship VirtIO drivers, so
  the installer shows "no drives found" against a virtio qcow2 disk. During install, load the
  storage driver from the VirtIO ISO at `\viostor\w11\amd64` (select the VirtIO SCSI
  controller) to make the disk visible.

## Network

| Setting | Value |
|---|---|
| Address | 192.168.10.100 (set as a **static** IP inside Windows) |
| Gateway | 192.168.10.1 (pfSense lab-corp) |
| DNS | **192.168.10.10 (DC01)** |

DNS must point at the domain controller, not pfSense or an external resolver. Domain-joined
Windows machines locate the DC through DNS SRV records that only the DC's DNS service knows
about; pointing DNS elsewhere makes the domain join fail.

Discrepancy to reconcile: `00-networks.md` lists WS01 as a DHCP reservation, but it was built
with a static address inside Windows. Either is workable; update the network map to say static,
or convert WS01 to a pfSense reservation, so the two docs agree.

## Accounts on this host

| Account | Type | Notes |
|---|---|---|
| `localadmin` | Local | Created during OOBE, used once; not part of the domain |
| `CORP\Administrator` | Domain admin | Used to perform the domain join |
| `CORP\alice.miller` | Domain user (HR OU) | Normal non-admin user; profile created by signing in once on WS01. Password `Pa$$w0rd!` (intentionally weak, present in rockyou). This is the foothold identity for the attack chain, not one of the four seeded misconfigurations. |

## Domain join

- Renamed to `WS01`, then joined to `corp.lab.lan` with `CORP\Administrator`.
- Verified before joining with `nslookup corp.lab.lan` (returns 192.168.10.10) and
  `Test-NetConnection 192.168.10.10 -Port 389` (LDAP reachable).

## Monitoring state

No Wazuh agent and no Sysmon yet. WS01 is a primary Phase 2 target: Wazuh agent enrollment,
Sysmon deployment (via GPO), and PowerShell logging all land on this host in Phase 2.

## Snapshots

| Name | State captured |
|---|---|
| `clean-domain` | Windows 11 joined to `corp.lab.lan`, `alice.miller` profile created |

## Issues hit during this build

- "No drives found" at install until the VirtIO storage driver was loaded from the drivers ISO.

## Screenshots captured

- WS01 logged in as `CORP\alice.miller`, and `nslookup corp.lab.lan` resolving to the DC,
  saved to `diagrams/raw/`.

## Commit reference

Documented with: `feat(workstation): build WS01 Windows 11 client, join corp.lab.lan, create
alice.miller foothold user`.
