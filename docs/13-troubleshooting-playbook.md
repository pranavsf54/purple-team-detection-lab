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
