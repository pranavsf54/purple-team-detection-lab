# Linux target (LINTGT01) build notes

As-built record of the Ubuntu Linux target: its domain join via SSSD, the DVWA web app, and the
deliberately weak local account. The step-by-step lives in
`docs/04-phase-1-infrastructure.md` Section 6; this file is the distilled record.

## Role in the lab

LINTGT01 is the domain-joined Linux server. It exists for two reasons: to produce Linux
telemetry (auditd, syslog, web access logs) alongside the Windows telemetry, which makes the
detection work and the portfolio more complete than a Windows-only lab; and to give the
attacker a Linux attack surface — an SSH service with a weak account, and a deliberately
vulnerable web app.

## Version and OS

- Ubuntu Server 22.04 LTS.
- `--os-variant ubuntu22.04`.

## Resources

| Setting | Value |
|---|---|
| RAM | 2 GB (2048 MB) |
| vCPU | 1 |
| Disk | 40 GB qcow2, thin provisioned, virtio bus |
| Networks | one NIC on `lab-corp` |

## Network

| Setting | Value |
|---|---|
| Address | 192.168.10.20/24 (static, set in the Subiquity installer) |
| Gateway | 192.168.10.1 (pfSense lab-corp) |
| DNS | **192.168.10.10 (DC01)** — required so `realm discover/join` can resolve the AD SRV records |
| Hostname | `lintgt01` |
| Local admin | `labadmin` (created during install); OpenSSH server enabled |

## Time synchronization

chrony installed and pointed at the pfSense lab-corp gateway (192.168.10.1), replacing the
default Ubuntu pool. Verify with `chronyc sources` that the 192.168.10.1 source is selected,
for the same Kerberos clock-skew reason that applies to every domain-joined host.

## Domain join (SSSD)

Joined to `corp.lab.lan` with realmd/SSSD rather than the Windows-style join:

```bash
sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss \
    adcli samba-common-bin oddjob oddjob-mkhomedir packagekit
sudo realm discover corp.lab.lan      # confirms it is an AD domain, kerberos-member
sudo realm join -U Administrator corp.lab.lan
```

Verified with `realm list` (shows the domain) and `id corp.lab.lan\alice.miller` (resolves the
AD user's UID/groups, confirming SSSD lookups work).

Login restriction (gotcha): a realm join permits no domain users by default, so logins must be
explicitly allowed. `alice.miller` was permitted:

```bash
sudo realm permit alice.miller@corp.lab.lan
```

SSH login from the host (on 192.168.10.2 via the corp veth) uses the long realm-qualified
username; home directory is auto-created by `oddjob-mkhomedir`:

```bash
ssh alice.miller@corp.lab.lan@192.168.10.20   # AD password Pa$$w0rd!
```

## Attack surface on this host

Two deliberate weaknesses, both lab-only:

- **DVWA (Damn Vulnerable Web App)** at `http://192.168.10.20/DVWA/` (login `admin` / `password`),
  served by Apache/PHP/MariaDB. The DVWA DB user is `dvwa` / `p@ssw0rd`. This is the HTTP attack
  surface.
- **Weak local SSH account** `svc-backup` / `summer2025` with passwordless sudo
  (`/etc/sudoers.d/svc-backup`, `NOPASSWD: ALL`). This simulates a poorly secured service
  account: trivial credential discovery followed by instant local privilege escalation once an
  attacker has a shell.

Naming caution: the local Linux account here is `svc-backup` (on LINTGT01). It is **not** the
same as the AD GPP credential `corp\backup-svc` documented in the domain controller notes. The
names are similar but the accounts, hosts, and attacks are different — do not conflate them in
the engagement report.

## Monitoring state

No Wazuh agent yet. LINTGT01 is a Phase 2 target for the Linux Wazuh agent, auditd
integration, and Apache log ingestion.

## Snapshots

| Name | State captured |
|---|---|
| `ready` | Domain joined, Apache + DVWA running, weak SSH user `svc-backup` present |

## Issues hit during this build

- None blocking recorded. Watch the realm-permit step: without it, a correct join still rejects
  every domain login, which reads like a broken join.

## Screenshots captured

- `realm list` output and the DVWA login page, saved to `diagrams/raw/`.

## Commit reference

Documented with: `feat(linux-target): build LINTGT01, join corp.lab.lan via SSSD, deploy DVWA
and weak svc-backup account`.
