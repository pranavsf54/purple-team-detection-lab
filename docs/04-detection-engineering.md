# Detection engineering process

> Filled in during Phase 3 once the first detection workflow runs end to end.
> This stub captures the methodology so future detections follow a consistent process.

## The detection development loop

Every detection in this lab follows the same eight-step loop:

1. Pick a MITRE ATT&CK technique to target.
2. Hypothesize: write down which log source and event ID should reveal this attack, before attacking.
3. Execute the attack from Kali (or where the technique requires).
4. Collect the telemetry that actually fired. Compare to the hypothesis.
5. Check whether Wazuh's built-in rules caught it, and how accurately.
6. Write a Sigma rule that expresses the detection logic.
7. Hand-translate the Sigma rule into a Wazuh local rule, validate with `wazuh-logtest`.
8. Re-run the attack, then run normal activity. Tune false positives. Document them.

## On Sigma to Wazuh: no automatic conversion exists

There is no working pySigma backend for Wazuh as of mid-2026, and the legacy `sigmac` tool is deprecated. Community converters exist but choke on aggregation and timeframe correlations, which are exactly the features Active Directory detections need.

The workflow this lab uses, treated as a deliberate design choice:

- **Sigma is the source of truth.** Every detection starts as a Sigma rule because Sigma is portable across SIEMs and demonstrates vendor-neutral detection engineering.
- **Hand-translate to Wazuh local rules.** Wazuh's own rule syntax supports the features we need for AD detections: `<frequency>`, `<timeframe>`, and `<if_matched_sid>` for thresholds and bursts.
- **Validate with `wazuh-logtest`.** Wazuh ships a command-line tool (`/var/ossec/bin/wazuh-logtest`) that pipes a raw log event through the rule engine and shows which rule matched. Every Wazuh rule gets at least one positive test (an event that should match) and one negative test (a legitimate-looking event that should not match) before it is considered done.
- **CI runs both checks.** A GitHub Actions workflow (added in Phase 2) lints all Sigma rules with `sigma check` and runs every Wazuh rule against its test events on every commit. A failed test blocks merge.

This workflow is the senior signal: brittleness versus robustness in detections is exactly what the "rule tuning" half of detection engineering is about, and a CI harness proves it isn't a one-time exercise.

## On tuning

The single most important step is step 8. Writing a rule that fires once on the exact attack command is easy. Writing a rule that fires on legitimate variations of that attack and stays silent on legitimate admin activity is the actual job. Every per-incident write-up in [06-incidents/](06-incidents/) documents the false positives encountered and how the rule was narrowed.

## MITRE ATT&CK coverage

The coverage script in `scripts/coverage-reporter/coverage.py` (Phase 6) parses every Sigma rule's `tags` field, extracts MITRE technique IDs, and emits a MITRE ATT&CK Navigator JSON layer. The output is in [05-mitre-coverage.md](05-mitre-coverage.md).