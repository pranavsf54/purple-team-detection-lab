# Detection engineering process

> This document will be filled in Phase 3 when the first detection workflow runs end to end.

See the process template in `15-documentation-guide.md`. The short version: Sigma is the source of truth, hand-translated to Wazuh frequency rules (no working Sigma-to-Wazuh converter exists), validated with `wazuh-logtest` against captured raw events, then tuned against legitimate activity until false positives are eliminated.