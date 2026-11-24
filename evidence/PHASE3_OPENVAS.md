# Phase 3 — Vulnerability Management (Greenbone OpenVAS)

## Objective
Add a dedicated vulnerability-management capability to the home lab: deploy **Greenbone
Community Edition (OpenVAS)**, run an authenticated/unauthenticated scan against the lab target,
and produce verifiable findings. This backs the "vulnerability management (Greenbone/OpenVAS)"
CV claim with real, reproducible hands-on work.

## Architecture
A third VM was added to the isolated UTM **Shared NAT** network (never bridged):

| Component | Detail |
|---|---|
| VM | **GREENBONE** (Ubuntu 24.04 Server, ARM64) |
| IP | 192.168.64.7 |
| Resources | 8 GB RAM / 4 vCPU / 100 GB disk |
| Runtime | Docker 29.1 + Compose 2.40 |

Greenbone Community Edition runs as a Docker Compose stack (19 containers):

- `gvmd` (Greenbone Management daemon, v26.43.0 / GMP 22.7)
- `openvasd` + `ospd-openvas` (22.10.6) — the scanner engine
- `pg-gvm` — PostgreSQL databases (feed + CVE/SCAP data, ~27 GB)
- `gsad` + `gsa` — REST API + web UI, served by `nginx` on :443 (HTTPS)
- Feed containers: `vulnerability-tests`, `scap-data`, `cert-bund-data`,
  `dfn-cert-data`, `data-objects`, `report-formats`, `notus-data`, `gpg-data`

Feed release: **24.10** · VT cache: **202611230547**.

Web UI: `https://192.168.64.7/` (default credentials `admin`/`admin` — change after first login).

## Setup (reproducible)
```bash
# Provision VM via UTM + cloud-init seed ISO (same recipe as Phase 1/2 VMs)
# Grow root to 100 GB (OpenVAS CVE database needs > 40 GB)
sudo apt-get install -y docker.io docker-compose-v2
cd ~/greenbone
curl -fsSL -o compose.yaml https://greenbone.github.io/docs/latest/_static/compose.yaml
# expose web UI to the lab subnet
sed -i 's/127\.0\.0\.1:443:443/443:443/' compose.yaml
sudo docker compose up -d
# wait for feed sync to complete (scan configs appear when done)
```

## Scan
- **Target:** UBUNTU-TARGET (192.168.64.6)
- **Config:** "Full and fast" (`daba56c8-73ec-11df-a475-002264764cea`)
- **Scanner:** "OpenVAS Default" (`08b69003-5fc2-4037-a479-93b440211c73`)
- **Port list:** All IANA assigned TCP
- **Result:** 23 results · completed 2026-11-24 00:11 UTC

## Findings (severity summary)
| Severity | Count |
|---|---|
| High / Critical | 0 |
| Medium | 0 |
| Low | 3 |
| Info / Log | 20 |

Notable findings on 192.168.64.6:
- **22/tcp SSH** — weak MAC algorithms supported (Low), PQC KEX algorithms
  supported, version/algorithm enumeration (Info)
- **general/tcp** — TCP timestamps information disclosure (Low)
- **general/icmp** — ICMP timestamp reply information disclosure (Low)
- **4444/tcp** — open (simulated backdoor service created in Phase 1/2)
- OS detection, hostname determination, CPE inventory (Info)

No critical/high findings — expected for a freshly patched Ubuntu host; the value
is the fully operational, repeatable scanning capability and the triage workflow.

## Next
- Windows agent + Sysmon (needs a Windows ISO/licence)
- Active Directory hardening phase
