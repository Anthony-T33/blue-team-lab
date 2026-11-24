# How to Run the Lab

This page is the quick way to bring the lab back up and re-demonstrate each
capability (SIEM, vuln scanning, AD hardening, Windows endpoint telemetry).

## 1. Bring up the VMs

| VM | Hypervisor | IP | Role |
|---|---|---|---|
| UBUNTU-SIEM | UTM | 192.168.64.5 | Wazuh manager + Elasticsearch + Kibana |
| UBUNTU-TARGET | UTM | 192.168.64.6 | Wazuh agent (attacked endpoint) |
| GREENBONE | UTM | 192.168.64.7 | Greenbone CE / OpenVAS (vuln scanning) |
| DC01 | UTM | 192.168.64.8 | Samba AD DC — domain `lab.local` |
| WIN11 | Parallels | 10.211.55.3 | Windows 11 ARM64 — Sysmon + Wazuh agent |

- **UTM** VMs use **Shared NAT** (`192.168.64.0/24`). Start them from UTM's window.
- **WIN11** needs **Parallels** (UTM cannot boot the Windows 11 ARM64 installer).
- SSH into Linux VMs with the lab key: `ssh -i ~/.ssh/lab_siem_key ubuntu@<ip>`.
- Start order matters only for dependencies: start `DC01` and `UBUNTU-SIEM` first,
  then the rest.

## 2. Demonstrate each capability

### SIEM monitoring (Phases 1–2)
- Wazuh dashboard: `https://192.168.64.5` (Kibana/Wazuh UI).
- Trigger the brute-force rule by failing SSH logins at `UBUNTU-TARGET` 6+ times in
  120 s; watch rule `100100` fire in `wazuh-alerts-*`.
- Custom rules live in `rules/local_rules.xml`.

### Vulnerability management (Phase 3)
- Greenbone UI: `https://192.168.64.7` (start the GREENBONE VM first).
- Run a "Full and fast" scan against `UBUNTU-TARGET` (`192.168.64.6`).
- See `evidence/PHASE3_OPENVAS.md` for the original 23 results + reproduction.

### Active Directory hardening (Phase 4)
- DC: `dc01.lab.local` (`192.168.64.8`), domain `LAB.LOCAL`.
- Check the hardened policy:
  `sudo samba-tool domain passwordsettings show`
  (min length 14, max age 60, lockout threshold 5).
- See `evidence/PHASE4_AD.md`.

### Windows endpoint + Sysmon → SIEM (Phase 5)
- Start WIN11 in Parallels, confirm it is domain-joined and streaming:
  - `nltest /dsgetdc:lab.local`
  - `sc query sysmon64a` (Sysmon running, ARM64)
  - On the SIEM: `sudo /var/ossec/bin/agent_control -l` → agent `002/WIN11` `Active`.
- Sysmon events land in the SIEM for correlation. See `evidence/PHASE5_WINDOWS_SYSMON.md`.

## 3. Networking notes (non-obvious gotchas)

These bit us during the build and are worth knowing:

1. **DNS must point at the DC.** On WIN11, the adapter's DNS is `192.168.64.8`,
   with **IPv6 disabled** — otherwise Parallels injects `prl-local-ns-server.shared`
   via router advertisements and silently breaks domain resolution.
2. **Keep clocks in sync.** Kerberos fails ("domain could not be contacted") if the
   DC and clients drift > 5 min. The UTM DC syncs via `systemd-timesyncd`.
3. **Wazuh enrollment group** must be a valid name (`default`), not arbitrary strings.

## 4. Credentials

Credentials are **lab-only** and are deliberately **not stored in this repo**.
See the Phase evidence files and your local provisioning notes for usernames/passwords.

## 5. Reproduce from scratch

The evidence files (`evidence/PHASE*.md`) contain the exact commands used at each
stage — provisioning, rule deployment, scan config, and the Windows domain-join +
Sysmon + Wazuh-agent steps — so the whole lab is reproducible end-to-end.
