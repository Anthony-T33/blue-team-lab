# Blue Team Security Lab

A hands-on, self-built home SIEM lab for practising **detection engineering, threat
hunting, and incident triage** — the day-to-day work of a SOC / blue-team analyst.

Everything runs locally on Apple silicon using UTM virtual machines. All detection is
real, reproducible, and evidence-backed (see `evidence/`).

## Architecture

```
[ UBUNTU-SIEM ]  — Wazuh manager 4.14.10 + Elasticsearch 8.19.25 + Kibana 8.19.25
      |            (the "brain": collects, normalises, correlates, alerts)
      |  TCP 1514 / Filebeat -> Elasticsearch (wazuh-alerts-* index)
      |
[ UBUNTU-TARGET ] — Wazuh agent (the "endpoint": brute-forced, tampered, backdoored)
```

- Hypervisor: UTM (native arm64, Shared NAT — never bridged).
- Both VMs are Ubuntu Server 24.04 (arm64), SSH access via key, sudo NOPASSWD.

## What's built (Phase 1 & 2)

**Phase 1 — SIEM core**: Elasticsearch (single node, security off for isolated lab),
Kibana, and the Wazuh manager, wired so agents' logs and security events flow into
Elasticsearch for searchable, correlated alerting.

**Phase 2 — endpoint detection + custom rules**: a second VM runs a Wazuh agent and
three custom detection rules were written and validated against simulated attacks:

| Rule ID | Name | Detects | MITRE ATT&CK |
|---------|------|---------|--------------|
| `100100` | Brute force | 6+ failed SSH logins in 120s (same source IP) | T1110 |
| `100200` | Integrity change | File modified under `/etc/` | T1565 |
| `100300` | Suspicious process | Netcat backdoor executed from `/tmp/.malware` | T1059 |

### Attack simulation
- **Brute force** — repeated `ssh` with non-existent users → rule 100100.
- **Tampering** — appended C2 hostnames to `/etc/hosts`, dropped a file in `/etc` → rule 100200.
- **Backdoor** — renamed `/bin/nc` to `/tmp/.malware/backdoor`, listening on 4444 → rule 100300 (via the command wodle).

## Repository layout

```
rules/local_rules.xml              # the 3 custom Wazuh detection rules
config/agent-wodle-command.xml     # suspicious-process monitoring (wodle command)
evidence/PHASE1_CHECKPOINT.md      # SIEM core bring-up evidence (versions, health, URLs)
evidence/PHASE2_FINDINGS.md        # alert triage + verdicts + MITRE mapping
```

## Reproduce

1. Deploy the rules: copy `rules/local_rules.xml` to the manager's `/var/ossec/etc/rules/`
   and restart `wazuh-manager`.
2. Add `config/agent-wodle-command.xml` to the agent's `ossec.conf` and restart `wazuh-agent`.
3. Fire the simulations above and confirm alerts in Kibana (index `wazuh-alerts-*`).

> No credentials, private keys, or personal data are stored in this repository.
