# Troubleshooting playbook

Repo path: `docs/13-troubleshooting-playbook.md`

A running record of problems hit during the build, the diagnosis path, and the fix. Each entry
is written so a future rebuild does not lose the same hours twice. Entries are grouped by
layer.

---

## Virtualization / libvirt

### Internal snapshot fails on UEFI VMs

**Symptom**

```
error: Operation not supported: internal snapshots of a VM with pflash based
firmware require QCOW2 nvram format
```

**Cause.** The VM uses OVMF (UEFI) firmware with a separate pflash NVRAM file. libvirt cannot
take an atomic internal snapshot that includes a raw NVRAM store.

**Fix (recommended for this lab): external snapshots.** They handle NVRAM correctly and are
simpler to reason about than converting firmware formats. Shut the guest down first for a clean
state, then:

```bash
sudo virsh shutdown dc01
sudo virsh snapshot-create-as dc01 base-install "description" --disk-only --atomic
```

**Alternative: convert the NVRAM to qcow2.** This enables internal snapshots but has sharp
edges that cost real time here:

- Convert the **VARS** template, never the **CODE** file. Converting the firmware executable
  (`OVMF_CODE.secboot.4m.fd`) and pointing NVRAM at it causes OVMF to execute variable data as
  code and crash at boot with `X64 Exception Type - 06 (#UD - Invalid Opcode)`.
- After converting, the XML `<nvram>` element must say `format='qcow2'`. Leaving the stale
  `format='raw'` makes OVMF misread the qcow2 header and crash.
- On a VM with `<os firmware='efi'>`, libvirt auto-manages the firmware block and rewrites the
  `<nvram>` line on every `virsh edit`, regenerating `template`/`templateFormat`. To stop that,
  either edit `/etc/libvirt/qemu/<vm>.xml` directly and `virsh define` it, or remove
  `firmware='efi'` from the `<os>` tag so libvirt stops auto-selecting firmware.
- A fresh-converted VARS store has no boot entries, so the guest lands in the EFI shell. Re-add
  the Windows entry with `bcfg boot add 0 FSx:\EFI\Microsoft\Boot\bootmgfw.efi "Windows Boot Manager"`.

Lesson: for a lab with frequent snapshots across attack/detect cycles, external `--disk-only`
snapshots avoided this entire class of problem.

---

## Networking / host access

### "RTNETLINK answers: File exists" when creating the host veth

**Symptom.** Re-running the veth setup commands after a reboot, or on top of an existing pair,
errors out.

**Cause.** The veth pair already exists (or a stale half survives).

**Fix.** Tear the pair down before recreating — deleting one end removes both:

```bash
sudo ip link del virhost-mgmt
```

Then recreate, or better, drive it from a single systemd unit and use
`systemctl restart lab-veth.service` instead of running the commands by hand. The pairs do not
survive a host reboot unless a unit recreates them.

### Wazuh dashboard unreachable from the host

**Symptom.** `https://192.168.56.10` does not load from the CachyOS host.

**Root cause (two compounding).** The VM was not holding the expected `192.168.56.10` address
(a DHCP lease from the OPT1 pool instead of the intended static netplan address), and the host
veth was in a stale state.

**Fix.** Confirm the VM's address is a real static netplan config, not a pool lease; recreate
the mgmt veth cleanly (see above); confirm a mgmt firewall rule permits the host segment.

---

## Time synchronization

### pfSense returns zero NTP replies

**Symptom.**

```
chronyc ntpdata 192.168.56.1   # Total RX 0
```

**Cause.** pfSense-side, not the client. The NTP service was not answering on the mgmt
interface.

**Fix path.** On pfSense, confirm the NTP service is enabled and bound to the OPT1 (mgmt)
interface, and that no firewall rule blocks UDP/123 inbound on that interface. Then verify from
the client:

```bash
chronyc sources -v   # the 192.168.56.1 row must show ^*
```

This **must** be working before DC01 is promoted: Active Directory uses Kerberos, and Kerberos
rejects tickets when clocks drift more than five minutes apart.

### chrony config path differs on newer Ubuntu

On 24.04+ the source config lives in `/etc/chrony/sources.d/` rather than `chrony.conf`. Not
relevant on the final 22.04 Wazuh build, but it cost time on the 24.04 attempt and is recorded
so the next person does not chase it.

---

## Tooling install (Kali)

### "Unable to locate package bloodhound-ce" / "kerbrute"

**Cause.** Neither name is a Kali apt package.

**Fix.**

- BloodHound: install `bloodhound` (this is now Community Edition) plus `bloodhound-ce-python`
  for the collector. Do not use legacy `bloodhound-python` / `bloodhound.py` — its output will
  not ingest into CE.
- kerbrute: not packaged. Install the GitHub release binary:

```bash
curl -L -o kerbrute https://github.com/ropnop/kerbrute/releases/latest/download/kerbrute_linux_amd64
chmod +x kerbrute && sudo mv kerbrute /usr/local/bin/
```

See `docs/03-build/04-attacker-kali.md` for the full toolset.

---

## Endpoint telemetry (Sysmon via GPO)

Deploying Sysmon to the domain workstations with the sysmon-modular config turned into a
layered debugging session, where fixing one cause exposed the next. Sysmon (System Monitor)
is a Microsoft Sysinternals tool that writes detailed process, network, and file events to
the Windows Event Log; without it, the default Windows logs are too thin for real detection
work. Each distinct problem is recorded separately below, because each has its own symptom
and its own fix, and a future rebuild will likely hit them in the same order.

### Sysmon kernel driver silently blocked by Smart App Control

**Symptom.** The GPO ran, the install script executed, and the log showed it reaching the
Sysmon install line, but Sysmon never appeared as a running service and no Sysmon events
arrived in Wazuh. No obvious error in the script output.

**Diagnosis.** Sysmon installs a kernel-mode driver to capture its telemetry. On a clean
Windows 11 image, Smart App Control (SAC) was enabled. SAC is a Windows 11 protection that
blocks apps and drivers it does not consider trusted, and it does so quietly, which is why
the failure looked like nothing happening rather than a clear "blocked" message.

**Fix.** Disable Smart App Control. Note the catch: once SAC is turned off it cannot be turned
back on without reinstalling Windows, because SAC only re-enables from a fresh install in
evaluation mode. So this is a one-way door, and it is the reason SAC should be handled at the
very start of building a Windows 11 lab image rather than discovered mid-deployment. After
disabling SAC, the driver installed and Sysmon events began flowing into Wazuh.

### AppLocker blocking the GPO startup script

**Symptom.** A GPO startup script that should have run the Sysmon install did not run at all.

**Diagnosis.** AppLocker (a Windows application allow-listing feature) had script rules
enabled, and those rules silently blocked the script from executing. AppLocker enforcement
on scripts is easy to forget because it fails closed and quietly.

**Fix.** Add an AppLocker path exception for the SYSVOL location the script runs from. SYSVOL
is the domain-wide shared folder on the Domain Controller that every machine can read, which
is why GPO scripts are staged there.

### Zone.Identifier alternate data stream causing silent execution failure

**Symptom.** Even with the script allowed to run, the downloaded Sysmon binary failed to
execute cleanly, again with no useful error.

**Diagnosis.** Files downloaded from the internet carry a Zone.Identifier alternate data
stream, a small hidden tag Windows attaches that marks the file as "came from the internet."
Windows treats tagged files as untrusted and can block or interfere with their execution. The
Sysmon binary staged into SYSVOL still carried this tag.

**Fix.** Unblock the file so the Zone.Identifier stream is removed before it is deployed, for
example with `Unblock-File` in PowerShell. Any binary deployed via SYSVOL should be unblocked
as part of staging it.

### GPO Preferences scheduled task beats a startup script for this

**Symptom.** Even once the above were fixed, the GPO startup-script approach was fragile and
hard to make reliable across reboots.

**Diagnosis.** For a task that needs to persist and run on boot, a GPO startup script is the
wrong tool. A GPO Preferences scheduled task is more reliable. The trap is that the Group
Policy Management Console GUI creates an `ImmediateTaskV2` by default, which runs once
immediately rather than persisting across reboots.

**Fix.** Author the scheduled task XML by hand as a `TaskV2` with a `BootTrigger`, so the task
is persistent and fires on every boot. The install script lives in SYSVOL and logs to a known
path so its success or failure can be checked after each boot. This was the approach that
finally produced reliable Sysmon deployment with events confirmed in Wazuh.
