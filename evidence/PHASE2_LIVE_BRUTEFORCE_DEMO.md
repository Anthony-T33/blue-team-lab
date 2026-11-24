# Live Detection Demo — Brute-Force SSH Attack Caught by Wazuh

**Date:** 2026-11-24
**Result:** ✅ SUCCESS — custom Wazuh rule `100100` fired in real time.

## Objective

Prove the home SIEM detects and alerts on a real (simulated) attack, end-to-end:
generate failed logins → Wazuh correlates → high-severity alert → analyst investigates.

## The attack (simulated)

From the attacker machine (a Mac at `192.168.64.1` on the lab's Shared NAT):

```bash
for i in 1 2 3 4 5 6 7 8; do ssh -o BatchMode=yes -o ConnectTimeout=4 hacker@192.168.64.6; done
```

8 connection attempts as the non-existent user `hacker` against the TARGET VM
(`ubuntu-target`, `192.168.64.6`). Every attempt is rejected (`Permission denied`).

## Detection

| Field | Value |
|---|---|
| Rule | `100100` |
| Level | 10 (high severity) |
| Description | BRUTE-FORCE: 6+ failed SSH logins (non-existent user) in 120s from the same source IP |
| MITRE ATT&CK | T1110 — Brute Force |
| Agent | `ubuntu-target` |
| Source IP | `192.168.64.1` (attacker Mac) |
| Timestamp | `2026-11-24T19:27:39.066Z` |

The rule (`rules/local_rules.xml`) uses `frequency 6 / timeframe 120` on Wazuh rule
`5710` (invalid-user SSH attempt), grouped by `same_source_ip` — so 6+ failed logins
from one host within 2 minutes collapse into a single level-10 alert.

## Pipeline

`Wazuh manager (analysisd)` → `Filebeat` → `Elasticsearch` (index `wazuh-alerts-*`).

## How to reproduce / verify

In the SIEM's Kibana UI:

1. Open **Discover** → select the **`wazuh-alerts-*`** data view.
2. Time range: **Last 7 days**.
3. KQL query: `rule.id:100100`
4. Confirm a level-10 alert with source IP `192.168.64.1`.

## Analyst takeaway

An alert is only useful when the analyst can (1) confirm it fired, (2) identify the
source host, and (3) tie it back to a rule and a MITRE technique. That full loop is now
demonstrated and documented in this repo.
