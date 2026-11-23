# Home Security Lab — Phase 2 Findings & Alert Triage

Date: 2026-11-23
Analyst: (lab owner)
Environment: UBUNTU-SIEM (Wazuh manager 4.14.10 + Elasticsearch 8.19.25 + Filebeat 8.19.25),
             UBUNTU-TARGET (Wazuh agent, 192.168.64.6)

## Scope
Three custom detection rules were written (in /var/ossec/etc/rules/local_rules.xml) and
validated against real attack simulations run against the UBUNTU-TARGET endpoint.

## Custom rules deployed

| Rule ID | Level | Detection | Technique |
|---------|-------|-----------|-----------|
| 100100  | 10    | 6+ failed SSH logins (invalid user) in 120s, same source IP | MITRE T1110 Brute Force |
| 100200  | 9     | File modified under /etc/ | MITRE T1565 Data Manipulation |
| 100300  | 12    | Suspicious process (backdoor binary in /tmp/.malware) | MITRE T1059 Command & Scripting |

## Alert triage (verdicts)

1. **BRUTE-FORCE** (rule 100100, lvl 10)
   - Event: 6–12 "Invalid user hackN" SSH attempts from srcip 127.0.0.1 within 120s.
   - Verdict: TRUE POSITIVE (simulated brute-force / credential-stuffing attempt).
   - Response: block source IP at firewall / fail2ban.

2. **INTEGRITY / FIM** (rule 100200, lvl 9)
   - Event: /etc/hosts modified (C2 hostnames appended), /etc/realtimetest.txt added.
   - Verdict: TRUE POSITIVE (hosts-file tampering / persistence indicator).
   - Response: review diff, restore /etc/hosts, investigate account used.

3. **MALWARE / SUSPICIOUS PROCESS** (rule 100300, lvl 12)
   - Event: /tmp/.malware/backdoor -lvp 4444 (renamed netcat listening on 4444).
   - Verdict: TRUE POSITIVE (backdoor / reverse-shell listener).
   - Response: kill process, quarantine binary, check persistence (cron/systemd).

## Verification evidence
- Elasticsearch index wazuh-alerts-4.x-2026.11.23 contains the three rule.ids above.
- All three rules fire at their configured severity (levels 9, 10, 12).
- Pipeline verified end-to-end: agent -> manager -> Wazuh analysis -> Filebeat -> Elasticsearch.

## Notes / tuning
- FIM required real-time monitoring (<directories realtime="yes">/etc</directories>) —
  the periodic full scan did not flag changes on this build.
- Suspicious-process detection uses the Wazuh command wodle (pgrep -af backdoor), mapped to rule 100300.
