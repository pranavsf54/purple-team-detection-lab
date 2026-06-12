# Domain Controller (DC01) build notes

## Domain

- Forest: corp.lab.lan
- NetBIOS: CORP
- Functional level: Windows Server 2016
- DC: DC01.corp.lab.lan at 192.168.10.10

## Role in the lab

DC01 is the Active Directory brain of the lab and the single most important host in it.
Active Directory (AD) is Microsoft's directory service: the central database of every user,
group, computer, and permission in the Windows network, and the thing that decides who is
allowed to do what. The server that runs it is the Domain Controller. Almost every Phase 4
attack targets this box, because owning the Domain Controller means owning the domain, and
almost every detection in the project depends on the audit events this box produces. If the
DC is misconfigured or under-logged, whole phases silently fail, so its build is recorded
here in more detail than the others.

## Version and OS

- Windows Server 2022 Datacenter, Desktop Experience (the full GUI install, as opposed to
  Server Core which has no desktop).
- UEFI firmware.
- `--os-variant win2k22`.

## Install specifics (decision record)

Two driver details are worth recording because they cost time if missed when installing
Windows under KVM/QEMU. KVM presents disks and network cards as VirtIO devices, which are
paravirtualised (the guest knows it is a VM and talks to the host efficiently) rather than
emulating real hardware. Windows does not ship VirtIO drivers, so:

- The VirtIO storage driver has to be loaded during install, or the Windows installer shows
  no disk to install onto. The driver comes from the `virtio-win` ISO, attached as a second
  CD-ROM and pointed at during the "Load driver" step.
- The NetKVM network driver has to be loaded after install from the same `virtio-win` ISO,
  or Windows shows no network adapter at all. This is the gotcha: a freshly installed DC with
  no NIC looks broken until NetKVM is loaded, at which point the adapter appears.

## DNS ordering

The order in which DNS is configured matters because of a chicken-and-egg problem. Before
the box is promoted to a Domain Controller, no DNS server exists on it. After promotion, AD
DS installs its own DNS server that holds the records for `corp.lab.lan`.

- During install, DC01 points at an external resolver so it can reach the network.
- After the AD DS promotion completes, DC01's DNS is switched to point at itself
  (`127.0.0.1`), because it is now the authoritative DNS server for the domain. Every other
  domain-joined machine points at DC01 for DNS so they can find domain resources.

Pointing a Domain Controller at an external resolver after promotion is a classic
misconfiguration that breaks domain name resolution in subtle ways, so the self-reference is
deliberate.

## Time

In a Windows domain the Domain Controller is the authoritative time source: every
domain-joined machine synchronises its clock to the DC. The DC in turn synchronises upward,
in this lab to pfSense on the corp interface (`192.168.10.1`). This chain is what keeps the
whole domain within the five-minute window Kerberos requires; a clock that drifts past that
window starts rejecting authentication tickets, which looks like random login failures.

## Population

Populated by BadBlood (https://github.com/davidprowe/BadBlood), which created
approximately 2,500 users, hundreds of groups, and randomized ACLs across
the default Users container and several BadBlood-generated OUs.

## Seeded misconfigurations for the Phase 4 attack chain

### 1. Kerberoastable service account
- Username: `sqlsvc` (CN under IT OU)
- Password: `Summer2024!` (present in rockyou wordlist)
- SPN: `MSSQLSvc/sqlserver.corp.lab.lan:1433`
- Attack: T1558.003 Kerberoasting

### 2. AS-REP roastable account
- Username: `jsmith` (CN under HR OU)
- Password: `Welcome123!` (present in rockyou)
- Account option: "Do not require Kerberos preauthentication" enabled
- Attack: T1558.004 AS-REP Roasting

### 3. GPP cleartext password
- GPO: `Mapped Drive Setup` (linked at domain root)
- Stored credential: `corp\backup-svc` / `BackupP@ss123`
- Location: SYSVOL\corp.lab.lan\Policies\{GUID}\User\Preferences\Drives\Drives.xml
- Attack: T1552.006 GPP Password / NetExec gpp-password module

### 4. Weak ACL: GenericWrite
- Attacker user: HOLLIE_MANN
- Target user: BUDDY_MCCARTY
- Permission: GenericWrite on target object
- Attack: T1098 Account Manipulation / ACL abuse via BloodHound

## Custom OUs

Workstations, Servers, IT, HR, Finance, Executives

## Snapshots

A `clean-domain` snapshot was taken after the domain was promoted and BadBlood had run, but
before the four misconfigurations were seeded. This gives a known-good baseline to roll back
to if a later attack phase corrupts the directory, without having to rebuild and re-run
BadBlood from scratch.

## Audit policy and logging (Phase 2)

These settings turn on the audit events that later detections depend on. They are configured
in Phase 2 and recorded in full in the defense-baseline note (`07-defense-baseline.md`); the
summary here exists so the dependency is visible from the DC's own notes.

- **Audit Directory Service Access** (an Advanced Audit Policy setting) is enabled, and a
  SACL (System Access Control List, the audit cousin of a permission list: it says "log when
  these operations happen on this object") is set on the domain root. Together these cause
  Windows to generate Event ID 4662 when directory objects are accessed. This is what the
  Phase 4 DCSync detection fires on, and without it that detection has nothing to see.
- **PowerShell Script Block Logging** (Event ID 4104) and **Module Logging** are enabled via
  GPO, so the actual PowerShell commands attackers run are recorded rather than just the fact
  that PowerShell launched.

Both are confirmed flowing into Wazuh as part of the Phase 2 baseline.
