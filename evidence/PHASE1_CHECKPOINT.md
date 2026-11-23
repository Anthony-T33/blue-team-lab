# Phase 1 Checkpoint — SIEM Core on UBUNTU-SIEM (2026-11-23)

One native-ARM64 VM (`UBUNTU-SIEM`, UTM "Virtualize", shared-NAT only) running the SIEM core:
Elasticsearch + Kibana + Wazuh manager, with Wazuh indexing into Elasticsearch.

## Verified status
- Elasticsearch **8.19.25** (arm64) — cluster `home-lab-siem`, health **green**, 1 node
- Kibana **8.19.25** (arm64) — up at http://192.168.64.5:5601, data view **"Wazuh SIEM"** (pattern `wazuh-*`, time field `timestamp`) created
- Wazuh manager **4.14.10-1** (arm64) — indexer connector initialized for all `wazuh-states-*` + `wazuh-vulnerabilities-*` indices
- **14 wazuh indices** present in Elasticsearch
- **344 alerts** generated locally; e.g. rule 5502 (PAM login session) fired from failed `su` attempts (agent `000`/manager node)
- Services all active: `elasticsearch`, `kibana`, `wazuh-manager`

## VM/install summary
- VM: 8 GB RAM, 4 cores, 60 GB disk (grown via qemu-img; ext4 root auto-grown by cloud-init), 4 GB swap
- Elastic official apt repo (artifacts.elastic.co, stable, ARM64 builds)
- Wazuh package: direct deb `wazuh-manager_4.14.10-1_arm64.deb` (613 MB) + sha512 verified
- ES config: single-node discovery, security disabled (isolated home lab, no exposure), 2 GB heap,
  paths in /etc/default/elasticsearch + yml, vm.max_map_count=262144
- Wazuh ossec.conf: indexer host set to `http://127.0.0.1:9200`

## URLs (from this Mac)
- Kibana: http://192.168.64.5:5601   (Discover -> "Wazuh SIEM" data view)
- Elasticsearch: http://192.168.64.5:9200

## Notes for evidence integrity
- CV lines stay "self-study" until Phase 2 (agents + rules + triage) evidence lands and this lab is
  committed to GitHub `blue-team-lab`. CISSP exam 7 Dec 2026 — phased, time-boxed.
