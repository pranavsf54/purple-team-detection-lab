# Detection rule tests

This directory holds test cases for every Sigma rule and Wazuh local rule in the repository. The tests are run automatically on every push by GitHub Actions (`.github/workflows/detections.yml`).

## Structue
```text
tests/
├── README.md                  # This file
└── sample-events/             # Raw log events captured during attacks
    ├── kerberoasting/
    │   ├── positive.json      # Event that should match the rule
    │   └── negative.json      # Legitimate event that should not match
    └── [detection-name]/      # One folder per detection
```

## Adding a test for a new rule

1. During detection development, capture the raw event that triggered the attack. In Wazuh, the raw event is visible in the alert detail; copy the `full_log` field.
2. Save it under `tests/sample-events/<rule-slug>/positive.json`.
3. Capture or construct a legitimate-looking event (the same event type but from non-malicious activity) and save it as `negative.json`.
4. The CI pipeline will pipe each through `wazuh-logtest` and assert positive matches, negative does not.

## Why this exists

A detection that fires once on the exact command you ran is a demo. A detection that fires reliably across attacker variations and stays quiet on legitimate activity is engineered. The tests prove which one we built.
