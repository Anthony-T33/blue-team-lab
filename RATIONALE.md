# Home Lab — What I Built and *Why* (Interview Explainer)

This is the companion to the CV: the reasoning behind every part of the lab, so you
can explain it confidently in an interview rather than just listing tools.

## The one-sentence story

> "I built a mini Security Operations Centre at home to learn the same tools and
> workflows a real SOC analyst uses — collecting logs from several systems into a SIEM,
> writing my own detection rules, scanning for vulnerabilities, and hardening identity —
> so I could prove I can do the job, not just read about it."

Interviewers don't want to know *that* you used Wazuh. They want to know **why** you
chose it, **what problem** each piece solves, and **how it maps to their world**.

## The big picture: how a real SOC works, and how my lab mirrors it

Every SOC runs the same four-stage loop. My lab has all four stages:

| SOC stage | What it means | My lab equivalent |
|---|---|---|
| 1. **Collect** | Gather logs from every system | Wazuh agents on each endpoint |
| 2. **Correlate & detect** | Turn raw logs into alerts | Wazuh manager + my custom rules |
| 3. **Investigate** | Search and triage the alert | Kibana (querying Elasticsearch) |
| 4. **Remediate / harden** | Fix what you found | Greenbone vuln scans + AD hardening |

The data flows like this: **endpoint → agent → forwarder → central store → analyst**.

```
Wazuh agents ──► Wazuh manager (analysisd) ──► Filebeat ──► Elasticsearch ──► Kibana
```

## Component by component — what, real-world name, and why

### Wazuh (manager + agents) — the SIEM
- **What it is:** a Security Information & Event Management platform that ingests logs
  from all endpoints and raises alerts when something suspicious happens.
- **Real-world name:** Splunk, IBM QRadar, Microsoft Sentinel.
- **Why I chose it:** it's open source, free, and the most common "first SIEM" — so the
  concepts transfer directly to the commercial tools an employer uses.
- **The "why" it exists:** logs are useless if they're scattered across machines. A SIEM
  centralises them so patterns (like 6 failed logins) become visible.

### Elasticsearch — the log storage engine
- **Real-world name:** the search/index backend of an Elastic Stack; Splunk's indexer.
- **Why:** once you collect millions of events, you need a fast search engine to query
  them. Elasticsearch stores every alert and lets you search it near-instantly.

### Kibana — the analyst's screen
- **Real-world name:** the visual/query layer (Kibana; Splunk's Search app).
- **Why:** detection is pointless if a human can't investigate. Kibana is where you
  search raw events, filter by rule, and confirm an alert.

### Filebeat — the log forwarder
- **Real-world name:** Filebeat, Fluentd, Logstash, Cribl.
- **Why:** it's the "pipe" that reliably moves events from the SIEM manager into storage.
  Separating collection from storage is how real deployments scale.

### Wazuh agent — the endpoint collector
- **Real-world name:** Splunk Universal Forwarder, Elastic Agent, osquery.
- **Why:** something has to live on each machine to gather its logs and ship them out.
  Every SIEM has an equivalent lightweight agent.

### OpenVAS / Greenbone — the vulnerability scanner
- **Real-world name:** Tenable Nessus, Qualys, Rapid7 InsightVM.
- **Why:** monitoring tells you when you're *being attacked*; scanning tells you where
  you're *weak before* they attack. This is the "vulnerability management" half of
  defence — feeding a patch/risk register is exactly what a GRC or VM analyst does.

### Samba Active Directory (DC01) — the identity system
- **Real-world name:** Microsoft Active Directory (or Entra ID / Azure AD in the cloud).
- **Why:** almost every enterprise authenticates users through AD. It's the "crown
  jewel" — if an attacker compromises AD, they own the network. Learning to **harden** it
  (password policy, account lockout) is a core defensive skill.

### Sysmon (Windows 11) — deep endpoint telemetry
- **Real-world name:** Sysmon (free) or EDR products like CrowdStrike, SentinelOne,
  Microsoft Defender.
- **Why:** Windows' default logs miss a lot. Sysmon records detailed process, network and
  file events — the raw material a modern SOC uses to spot malicious behaviour.

### Custom Wazuh rules — "detection engineering"
- **Real-world name:** Sigma rules, Splunk correlation searches, "detection content".
- **Why:** the out-of-the-box rules are generic. Real SOCs write rules for *their*
  environment. Writing three (brute-force, file-integrity, suspicious-process) showed I
  understand how a detection is built, not just consumed.

### MITRE ATT&CK — the shared vocabulary
- **What it is:** a framework that catalogues adversary techniques.
- **Why:** I mapped each of my rules to a MITRE technique (e.g. T1110 = brute force).
  This is how professionals describe threats — and it's a keyword worth using in interviews.

## The five build phases, and WHY each mattered

| Phase | What | Why it's necessary |
|---|---|---|
| 1 — SIEM foundation | Wazuh + Elastic Stack + Filebeat + agents | Everything else is useless without centralised log collection first. |
| 2 — Custom rules | 3 rules + MITRE mapping | Alerts don't exist until someone defines "what counts as an attack". |
| 3 — Vuln scanning | Greenbone/OpenVAS scan | Defence needs to *find* weaknesses, not just detect active attacks. |
| 4 — AD hardening | Samba AD + strict password policy | Identity is the highest-value target, so it's hardened first. |
| 5 — Endpoint telemetry | Sysmon + Wazuh agent on Windows 11 | Mirrors the real endpoint (Windows) most attackers target. |

## The live demo, and why it's the important part

I ran a simulated SSH brute-force attack (8 wrong logins as a fake user) and watched my
SIEM raise a **level-10 alert in real time**, then triaged it in Kibana. Why this matters:

- It proves the pipeline is **real and working**, not theoretical.
- It shows the **full SOC loop**: attack → detect → investigate → document.
- It's **reproducible and evidenced** on GitHub, which is far stronger than "I studied X."

## Interview soundbites (how to say it out loud)

- **"Tell me about your lab."** → "It's a five-VM SOC environment: Wazuh + Elastic Stack
  for SIEM, OpenVAS for vulnerability scanning, and a Samba Active Directory domain I
  hardened. I wrote three detection rules mapped to MITRE ATT&CK and proved one by
  triggering a real brute-force alert."
- **"Why Wazuh?"** → "It's free and open source, but it teaches the same workflow as
  Splunk or Sentinel — log collection, correlation, alerting, and querying in Kibana."
- **"What did you actually *do*?"** → "I didn't just install it. I wrote the rules,
  triggered an attack, triaged the alert, scanned for vulnerabilities, and hardened the
  domain — then documented it all on GitHub."
- **"How does this help in a GRC/audit role?"** → "Vulnerability scans feed the risk
  register; the AD hardening is a control; and I documented everything to an auditable
  standard — which is the GRC discipline applied to technical work."
