# Home Blue-Team Security Lab — Build Write-Up

**Author:** Anthony Osaheni Omwanghe · **Repo:** github.com/AnthonyOmwanghe/blue-team-lab

## 1. Objective

Build a realistic, hands-on Security Operations lab at home to gain practical
experience in **SIEM monitoring, vulnerability management, and Active Directory
hardening** — the exact skills employers look for in SOC analysts, security
engineers, and GRC/audit professionals. Every step is documented and version-controlled
on GitHub so it doubles as verifiable evidence.

## 2. Architecture

Five virtual machines on two hypervisors, networked together to model a small
enterprise:

| VM | Hypervisor | Role | Key software |
|---|---|---|---|
| UBUNTU-SIEM | UTM | Central monitoring platform | Wazuh manager 4.14, Elasticsearch 8.19, Kibana 8.19, Filebeat |
| UBUNTU-TARGET | UTM | Attacked endpoint (victim) | Wazuh agent |
| GREENBONE | UTM | Vulnerability scanner | Greenbone Community Edition / OpenVAS |
| DC01 | UTM | Identity/Domain controller | Samba Active Directory (domain `LAB.LOCAL`) |
| WIN11 | Parallels | Windows workstation | Windows 11 ARM64, Sysmon, Wazuh agent |

**Data pipeline (the SIEM core):**

```
Wazuh agents ──► Wazuh manager (analysisd) ──► Filebeat ──► Elasticsearch ──► Kibana
   (endpoints)      (collects + correlates)       (ships)      (stores)         (visualise)
```

## 3. What each tool did

- **Wazuh (manager + agents)** — the Security Information and Event Management
  (SIEM) engine. Agents on every endpoint collect logs; the manager analyses them in
  real time and raises alerts using detection rules.
- **Elasticsearch** — the search engine that stores all security events and alerts.
- **Kibana** — the web UI used to search, filter and investigate alerts (the
  analyst's console).
- **Filebeat** — a lightweight shipper that forwards Wazuh alerts into Elasticsearch.
- **Greenbone / OpenVAS** — vulnerability scanner; probes a target and produces a
  report of weaknesses (unpatched software, open risky ports, misconfigurations).
- **Samba Active Directory (DC01)** — the identity system; manages users, passwords
  and access, and was hardened with a strict password/account policy.
- **Sysmon (on Windows 11)** — Microsoft's System Monitor; emits deep process, network
  and file telemetry that the Wazuh agent sends to the SIEM.

## 4. Custom detection rules (the "detection engineering" work)

Three custom Wazuh rules were written (`rules/local_rules.xml`) and mapped to MITRE
ATT&CK:

| Rule | Level | Detects | MITRE |
|---|---|---|---|
| 100100 | 10 | 6+ failed SSH logins from one IP in 120s (brute force) | T1110 — Brute Force |
| 100200 | 9 | File modified under `/etc` (integrity / tampering) | T1565 — Data Manipulation |
| 100300 | 12 | Suspicious backdoor process in `/tmp/.malware` | T1059 — Command & Scripting |

## 5. Build phases

1. **Phase 1 — SIEM foundation:** stood up Wazuh manager + Elasticsearch + Kibana +
   Filebeat on UBUNTU-SIEM, enrolled agents on the endpoints.
2. **Phase 2 — Detection engineering:** wrote and deployed the three custom rules and
   proved the brute-force rule fires.
3. **Phase 3 — Vulnerability management:** ran a Greenbone/OpenVAS scan against the
   target and documented the 23 initial findings.
4. **Phase 4 — Active Directory hardening:** built the Samba AD domain and enforced a
   hardened policy — minimum password length 14, maximum age 60 days, account lockout
   threshold 5.
5. **Phase 5 — Windows endpoint telemetry:** domain-joined Windows 11, installed Sysmon
   plus the Wazuh agent, and confirmed it was streaming into the SIEM.

## 6. Live demonstration (the proof it works)

A simulated brute-force attack was run against UBUNTU-TARGET:

```bash
for i in 1 2 3 4 5 6 7 8; do ssh -o BatchMode=yes -o ConnectTimeout=4 hacker@192.168.64.6; done
```

Within seconds, Wazuh correlated the failures and raised **rule 100100** (level 10). The
alert was investigated in Kibana, identifying the source IP (`192.168.64.1`). Full
evidence: `evidence/PHASE2_LIVE_BRUTEFORCE_DEMO.md`.

## 7. Real problems solved along the way

Building this required genuine troubleshooting, not just following a guide:

- **Kerberos clock skew** breaking domain authentication (fixed with NTP time sync).
- **Parallels injecting IPv6 DNS** that silently broke domain resolution (fixed by
  disabling IPv6 and pointing DNS at the domain controller).
- **Wazuh enrollment group naming** (invalid group names rejected).
- **Sysmon on ARM64** (required the native `Sysmon64a` build).

## 8. Skills demonstrated

- SIEM operations and log analysis (Wazuh + Elastic Stack + Kibana).
- Detection engineering and MITRE ATT&CK mapping.
- Vulnerability management (OpenVAS/Greenbone).
- Active Directory and identity hardening.
- Endpoint detection and response fundamentals (Sysmon).
- Linux/Windows administration at the command line.
- Documentation, version control, and evidence management (Git/GitHub).
