# Wireshark PCAP Deep Analysis Lab

Packet-level PCAP forensics: reconstructing attacks from raw capture alone, with no
SIEM safety net, then tying each finding back to SIEM and IDS detection logic. Built to
close the packet-analysis gap in a detection-engineering portfolio.

**Author:** William James ([@WiLL75G](https://github.com/WiLL75G)) · Detection engineer / SOC analyst

---

## What this lab demonstrates

Reading traffic at the packet level and answering the three questions a Tier 1 analyst
asks of any capture: who is the aggressor, what is the technique and did it succeed, and
what do I escalate with. Each scenario is captured victim-side (as a real sensor sees
it), analysed from the PCAP alone, then mapped back to a detection.

## Lab topology

| Host | Role | IP |
|---|---|---|
| Kali Linux | Attacker | 192.168.64.15 |
| Ubuntu Server (wazuh-manager) | Target / Linux telemetry | 192.168.64.12 |
| Windows 11 (JAMES-VM) | Target / capture host | 192.168.64.17 |
| macOS host | Splunk indexer + Wazuh manager (Docker) | — |

Capture tooling: Wireshark / tshark (victim-side). SIEM: Splunk. EDR: Wazuh. IDS: Suricata.

## MITRE ATT&CK coverage

| Technique | Scenario | Status |
|---|---|---|
| T1046 Network Service Discovery | 01 Port scan | Complete |
| T1110 Brute Force | 02 SSH brute force | Planned |
| T1040 Network Sniffing / cleartext creds | 03 Cleartext credentials | Planned |
| T1048 / T1071 Exfil & C2 | 04 DNS exfiltration | Planned |

## Repository contents

| Document | Purpose | Status |
|---|---|---|
| [SETUP.md](SETUP.md) | Capture environment build (Wireshark/tshark + Npcap on Windows) | Complete |
| [docs/TIER1-PLAYBOOK.md](docs/TIER1-PLAYBOOK.md) | Packet-analysis scenario strategy | Complete |
| [docs/FILTERS.md](docs/FILTERS.md) | Tier 1 Wireshark/tshark filter kit | Complete |
| [docs/SPLUNK-TIER1-PLAYBOOK.md](docs/SPLUNK-TIER1-PLAYBOOK.md) | Splunk triage mental model | Complete |
| [docs/TIER1-DRILLS.md](docs/TIER1-DRILLS.md) | Pattern-recognition practice program | Complete |
| [docs/TIER1-DAILY-PRACTICE.md](docs/TIER1-DAILY-PRACTICE.md) | Verified weekly drill rotation (live-data tested) | Complete |
| [analysis/01-portscan.md](analysis/01-portscan.md) | Port scan PCAP analysis (T1046) | In progress |
| analysis/02-ssh-bruteforce.md | SSH brute force (T1110) | Planned |
| analysis/03-cleartext-creds.md | Cleartext credential exposure (T1040) | Planned |
| analysis/04-dns-exfil.md | DNS exfiltration (T1048) | Planned |

## Status

Active build. Port-scan capture and the Tier 1 analysis/drill documentation are
complete; scenarios 02 through 04 are in progress.
