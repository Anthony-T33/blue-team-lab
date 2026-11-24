# Phase 5 — Windows 11 Endpoint + Sysmon → SIEM (Wazuh)

## Objective
Add a real Windows endpoint to the lab, join it to the `LAB.LOCAL` domain, run
**Sysmon** for rich endpoint telemetry, and ship Windows + Sysmon events into the
SIEM via the **Wazuh agent**. This backs the "hands-on detection capability" CV
claim with an end-to-end, reproducible pipeline.

## Why Parallels (not UTM) for Windows 11
The lab is Apple Silicon (ARM64). UTM/QEMU cannot boot the Windows 11 ARM64
installer (the firmware never exposes the ISO as a bootable filesystem). A free
**Parallels Desktop** trial / VMware Fusion is required; Windows 11 ARM64 installed
cleanly under Parallels.

## Architecture
| Component | Detail |
|---|---|
| VM | **WIN11** (Windows 11 Pro ARM64, build 26300) |
| Hypervisor | Parallels Desktop |
| IP | 10.211.55.3 (Parallels Shared NAT) |
| Resources | 8 GB RAM / 4 vCPU (64 GB virtual disk) |
| Membership | Domain-joined to `LAB.LOCAL` (DC01 @ 192.168.64.8) |
| Endpoint telemetry | **Sysmon v15.22** (ARM64 `Sysmon64a`), SwiftOnSecurity config |
| Agent | **Wazuh agent 4.14.10**, enrolled as agent `002`, group `default` |

## Cross-hypervisor networking
Parallels (NAT `10.211.55.0/24`) and UTM (Shared NAT `192.168.64.0/24`) are
different hypervisors. Connectivity works because Parallels NAT routes through the
macOS host, which also hosts UTM's shared bridge, so `10.211.55.3` → `192.168.64.x`
traverses the host. No bridging of the home LAN was used.

## Steps performed

### 1. Join the domain
- Routed the endpoint's DNS to the domain controller (`192.168.64.8`).
- `Add-Computer -DomainName lab.local -Credential LAB\Administrator`
- `Rename-Computer -NewName WIN11` (domain credential).

### 2. Install & configure Sysmon
```powershell
Invoke-WebRequest https://download.sysinternals.com/files/Sysmon.zip
Invoke-WebRequest https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml
Expand-Archive Sysmon.zip
sysmon64a.exe -accepteula -i sysmonconfig-export.xml   # native ARM64 build
```
- Driver/service: `Sysmon64a` (RUNNING), config schema 4.91.
- Result: `Microsoft-Windows-Sysmon/Operational` event log actively populated
  (hundreds of process/network/registry events).

### 3. Install & enroll the Wazuh agent
```powershell
Invoke-WebRequest https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi
msiexec /i wazuh-agent-4.14.10-1.msi /qn WAZUH_MANAGER=192.168.64.5 \
    WAZUH_REGISTRATION_SERVER=192.168.64.5 WAZUH_AGENT_NAME=WIN11
```
- Auto-enrollment against `wazuh-authd` (`use_password=no`).
- Added a `<localfile>` for `Microsoft-Windows-Sysmon/Operational` so Sysmon
  events are collected and forwarded.

## Issues resolved (real-world troubleshooting, documented as evidence of depth)

1. **DC clock skew (~15.5 h).** The UTM DC's clock was wrong, which breaks
   Kerberos (misleading *"domain could not be contacted"*). Fixed by re-syncing
   time and pointing `systemd-timesyncd` at a public NTP source.
2. **IPv6 DNS interference.** Parallels injects its own IPv6 DNS
   (`prl-local-ns-server.shared`) via router advertisements, overriding the DC DNS
   and breaking SRV/domain resolution. Fixed by disabling IPv6 on the adapter.
3. **Bad enrollment group.** `WAZUH_AGENT_GROUP=windows` was rejected by
   `authd` (*"Invalid group: windows"*); corrected to `default`.

## Verification
- `nltest /dsgetdc:lab.local` → DC `\\192.168.64.8`, role flags PDC/GC/KDC/LDAP/DNS.
- `Get-WmiObject Win32_ComputerSystem` → `Domain: lab.local, PartOfDomain: True`.
- Sysmon event log record count growing (process create, network connection, etc.).
- Wazuh manager `agent_control -l`:
  ```
  ID: 002, Name: WIN11, IP: any, Active
  ```
- Windows / Sysmon events landing in the SIEM for analysis against the Wazuh rules
  (see `rules/`).
