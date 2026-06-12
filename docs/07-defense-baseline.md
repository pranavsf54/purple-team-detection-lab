# Phase 2 defense-baseline build notes

This note records the Phase 2 work: getting telemetry flowing from every monitored host into
Wazuh, turning on the audit settings that later detections depend on, applying log retention,
and capturing the normal noise floor of the environment before any attacks are run. The point
of a baseline is simple: you cannot tell what an attack looks like in the logs until you know
what normal looks like first. Everything here is the "before" picture.

## Wazuh agent enrollment

The Wazuh agent is the small program installed on each monitored host that ships its logs and
local detections back to the Wazuh manager. Agents are enrolled on the three monitored hosts:

| Host | OS | Agent status |
|---|---|---|
| DC01 | Windows Server 2022 | enrolled, reporting (green) |
| WS01 | Windows 11 | enrolled, reporting (green) |
| LINTGT01 | Ubuntu 22.04 | enrolled, reporting (green) |

- Agent version: `__FILL IN: e.g. 4.14.5__`
- Enrollment method: `__FILL IN: e.g. manual agent-auth with the manager at 192.168.56.10__`
- Confirmation: all three show as Active in Agents management in the dashboard.

KALI01 is deliberately not enrolled. It is the attacker box on the isolated attacker segment,
and putting a Wazuh agent on it would be both unrealistic (a real attacker is not running your
agent) and would contaminate the telemetry the lab is trying to evaluate.

## Sysmon deployment via GPO

Sysmon (System Monitor) expands Windows logging well beyond the sparse defaults, recording
process creation, network connections, file and registry changes, and more. It was pushed to
the Windows hosts via Group Policy using Olaf Hartong's `sysmon-modular` config, the community
standard config that is broader than the older SwiftOnSecurity one.

The deployment was not smooth; the full saga (Smart App Control blocking the kernel driver,
AppLocker blocking the startup script, the Zone.Identifier stream, and the scheduled-task
approach that finally worked) is recorded in the troubleshooting playbook
(`docs/13-troubleshooting-playbook.md`). The short version: Smart App Control on the clean
Windows 11 image was silently blocking Sysmon's kernel driver, and disabling it was the fix.

**Confirmation that events actually arrive in Wazuh, not just that the service runs.** This is
the check that matters, because a running Sysmon service that is not forwarding is useless.
In the dashboard's Discover view, search for a Sysmon event to prove the pipeline end to end:

- Search used: `__FILL IN: e.g. data.win.system.providerName: "Microsoft-Windows-Sysmon"__`
- Example event seen: `__FILL IN: e.g. Event ID 1 (process creation) from WS01 at <time>__`
- Result: Sysmon events confirmed arriving from `__FILL IN: which hosts__`.

## PowerShell Script Block and Module Logging

Enabled via GPO on the Windows hosts. Script Block Logging records the actual PowerShell code
that runs as Event ID 4104, and Module Logging records pipeline execution details. Without
these, you see that PowerShell ran but not what it did, which is useless against attackers who
live in PowerShell. Confirmed flowing into Wazuh.

- Confirmation search: `__FILL IN: e.g. data.win.system.eventID: "4104"__`
- Result: `__FILL IN: 4104 events confirmed from <hosts>__`

## Audit Directory Service Access and the domain-root SACL

Enabled on DC01. This is the audit-critical Phase 2 step. "Audit Directory Service Access" is
an Advanced Audit Policy setting, and a SACL (System Access Control List, the audit version of
a permission list, meaning "log when these operations happen on this object") is set on the
domain root. Together they make Windows emit Event ID 4662 when directory objects are
accessed. The Phase 4 DCSync detection fires on 4662, so without this the detection has
nothing to see and would silently never fire. Cross-referenced in the DC build note
(`03-domain-controller.md`).

- Confirmation: `__FILL IN: 4662 events visible in Wazuh after a test directory read, e.g. an
  authorised replication or an ldap query against the domain root__`

## Log retention (ISM policy)

Wazuh stores events in its OpenSearch-based indexer, which will fill the disk if left
unbounded. An ISM (Index State Management) policy automates index rotation and deletion. The
policy at `wazuh/ism-retention-14d.json` deletes indices older than 14 days.

- Applied via: `__FILL IN: dashboard ISM UI, or the indexer REST API call used__`
- Confirmation it took effect: `__FILL IN: policy shows as attached to the wazuh-alerts
  index pattern in the ISM dashboard__`

## The 48-hour baseline noise floor

The baseline window runs the environment with normal activity (logons, background services,
BadBlood's population sitting idle, ordinary domain chatter) and no attacks, so the normal
volume and the normally-noisy rules are known before Phase 3 begins. Fill this in once the
window closes; do not estimate it, because the whole value of the baseline is that the numbers
are real.

- Window: from `__FILL IN: start timestamp__` to `__FILL IN: end timestamp__`
- Total events ingested: `__FILL IN__`
- Total alerts raised: `__FILL IN__`
- Noisiest built-in rules (rule ID, description, count):
  - `__FILL IN__`
  - `__FILL IN__`
  - `__FILL IN__`
- Rules already known to need tuning down in Phase 3, and why:
  - `__FILL IN__`

This noise-floor table is what makes the baseline a real deliverable rather than just a
waiting period, and it is the "before" half of every tuning story told later in the project:
"this rule fired N times an hour on normal activity, here is how I tuned it down to
high-fidelity alerts."
