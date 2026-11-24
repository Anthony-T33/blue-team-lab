# Home Lab — Hands-On Practice Runbook

A step-by-step guide to re-run the lab and practice the SOC workflow yourself. Each step
shows **What**, **Why**, and the **Real-world name** so you learn the job, not just the
clicks.

## Quick reference

| VM | Hypervisor | IP | Role |
|---|---|---|---|
| UBUNTU-SIEM | UTM | 192.168.64.5 | Wazuh + Elasticsearch + Kibana (SIEM) |
| UBUNTU-TARGET | UTM | 192.168.64.6 | The "victim" endpoint |
| GREENBONE | UTM | 192.168.64.7 | OpenVAS vulnerability scanner |
| DC01 | UTM | 192.168.64.8 | Samba Active Directory (domain `LAB.LOCAL`) |
| WIN11 | Parallels | 10.211.55.3 | Windows 11 + Sysmon + Wazuh agent |

- **SSH into the Linux VMs:** `ssh -i ~/.ssh/lab_siem_key ubuntu@<ip>` (passwordless sudo).
- **SIEM/Kibana UI:** `http://192.168.64.5:5601` (no login — security is off).
- **Greenbone UI:** `https://192.168.64.7` → user `admin` / password `admin`.
- **AD admin password:** use the one from your own lab notes (deliberately not stored here).

---

## PART A — Start the lab (the right order)

**Why order matters:** the SIEM and the domain controller are dependencies — agents can't
report if the SIEM isn't up, and domain login needs the DC running first.

### Step 1 — Start the SIEM (UTM)
- **What:** open UTM → start `UBUNTU-SIEM` → wait ~1–2 min.
- **Why:** everything else sends data *to* it.
- **Real-world name:** standing up your SIEM / log platform before agents.

### Step 2 — Start DC01 (UTM)
- **What:** start `DC01`.
- **Why:** it's the identity source; start it before the machines that join the domain.
- **Real-world name:** domain controller uptime / authentication services.

### Step 3 — Start UBUNTU-TARGET (UTM)
- **What:** start `UBUNTU-TARGET`.
- **Why:** this is the endpoint we'll "attack".
- **Real-world name:** a monitored host / asset.

### Step 4 — Start WIN11 (Parallels)
- **What:** in **Parallels** (not UTM), start the VM named **"Windows 11"**.
- **Why:** it's the Windows endpoint streaming Sysmon + Wazuh telemetry.
- **⚠️ Gotcha:** there's an old broken **"Win11"** in UTM — ignore it; use the Parallels one.

### Step 5 — Start GREENBONE (UTM)
- **What:** start `GREENBONE`.
- **Why:** the scanner (only needed when running a scan).

---

## PART B — Practice exercise 1: detect a brute-force attack (the main one)

**Goal:** trigger Wazuh rule **100100** and see the alert in Kibana.

**Real-world meaning:** SSH brute-force = attacker repeatedly guessing passwords = MITRE
**T1110**. This is one of the most common attacks, and a SOC analyst sees it constantly.

### 1. Play the attacker (from your Mac's Terminal — a *fresh* window, not SSH'd into a VM)
```bash
for i in 1 2 3 4 5 6 7 8; do ssh -o BatchMode=yes -o ConnectTimeout=4 hacker@192.168.64.6; done
```
- **What:** 8 login attempts as the non-existent user `hacker` against TARGET.
- **What you'll see:** `Permission denied` lines — that's the *point* (the attack is rejected, but every failure is logged).

### 2. Investigate the alert in Kibana
1. Browser → `http://192.168.64.5:5601`.
2. ☰ menu → **Discover** → select the **`wazuh-alerts-*`** data view.
3. Time range → **Last 7 days**.
4. Search: `rule.id:100100`
- **What you'll see:** a level-10 alert *"BRUTE-FORCE: 6+ failed SSH logins…"* with source IP `192.168.64.1` (your Mac).
- **Why this matters:** this is the exact "alert → triage → identify source" loop a SOC analyst performs.

---

## PART C — Practice exercise 2: vulnerability scan

**Real-world meaning:** vulnerability management — proactively finding weaknesses
before an attacker does. This output feeds a **risk register** (a GRC staple).

1. Browser → `https://192.168.64.7` (accept the self-signed warning) → `admin` / `admin`.
2. Create a new **Target** pointing at `192.168.64.6`.
3. Run a **"Full and fast"** scan against it.
4. When it finishes, open the **report** — you'll see findings with severities.
- **Why:** this is how a VM/GRC analyst decides what to patch and prioritise risk.

---

## PART D — Practice exercise 3: check the AD hardening (a control)

**Real-world meaning:** verifying a security *control* exists and is configured correctly —
something an auditor/GRC person literally asks for.

SSH into DC01:
```bash
ssh -i ~/.ssh/lab_siem_key ubuntu@192.168.64.8
sudo samba-tool domain passwordsettings show
```
- **What you're checking:** the password policy you hardened:
  - Minimum password length **14**
  - Maximum age **60 days**
  - Account lockout threshold **5**
- **Why:** a strict password + lockout policy is a core control against brute-force/credential attacks.

---

## PART E — Practice exercise 4: confirm Windows telemetry (Parallels)

**Real-world meaning:** verifying endpoint agents are actually streaming data to the SIEM.

In Windows 11, open **Command Prompt** (Win+R → `cmd`):
```cmd
sc query WazuhSvc
sc query sysmon64a
```
- **What you're checking:** both should show `STATE : 4 RUNNING`.
- **Why:** no running agent = no telemetry = blind spot. This is the first thing a SOC
  checks when an endpoint goes quiet.

---

## PART F — Advanced: the other two rules (try when comfortable)

- **Rule 100200 (integrity, MITRE T1565):** modify any file under `/etc` on TARGET, e.g.
  `ssh -i ~/.ssh/lab_siem_key ubuntu@192.168.64.6 "sudo touch /etc/tamper-test"`, then
  search `rule.id:100200` in Discover. This models detecting unauthorised file changes.
- **Rule 100300 (suspicious process, MITRE T1059):** simulates a backdoor binary in
  `/tmp/.malware`. Models detecting malware/command execution.

---

## After every session: document it

Pick *one* thing you demonstrated, write 3–4 lines of what happened + a screenshot, and
commit it to `blue-team-lab/evidence/`. That consistent habit is itself a GRC/audit
behaviour — and keeps building the public proof.
