# Wireshark PCAP Deep Analysis Lab

![Scenarios](https://img.shields.io/badge/scenarios-4-00b4d8)
![Focus](https://img.shields.io/badge/focus-packet%20analysis-1d3557)
![SIEM](https://img.shields.io/badge/SIEM-Splunk-f99d1c)
![IDS](https://img.shields.io/badge/IDS-Suricata-e63946)
![EDR](https://img.shields.io/badge/EDR-Wazuh-3a86ff)
![Status](https://img.shields.io/badge/status-complete-2a9d8f)

Packet level attack reconstruction. Four attacks captured victim side and analysed from raw packets alone, with no SIEM safety net, then tied back to SIEM and IDS detection logic. Built to close the packet analysis gap in a detection engineering portfolio.

Author: William James ([@WiLL75G](https://github.com/WiLL75G)) . Detection engineer and SOC analyst

---

### Tooling and coverage

![Wireshark](https://img.shields.io/badge/Wireshark-1679a7?logo=wireshark&logoColor=white)
![tshark](https://img.shields.io/badge/tshark-1679a7)
![tcpdump](https://img.shields.io/badge/tcpdump-004080)
![Splunk](https://img.shields.io/badge/Splunk-000000?logo=splunk&logoColor=white)
![Suricata](https://img.shields.io/badge/Suricata-e63946)
![Wazuh](https://img.shields.io/badge/Wazuh-3a86ff)
![Nmap](https://img.shields.io/badge/Nmap-4682b4)
![Hydra](https://img.shields.io/badge/Hydra-6a4c93)

![T1046](https://img.shields.io/badge/T1046-Recon-yellow)
![T1110](https://img.shields.io/badge/T1110-Brute%20Force-orange)
![T1040](https://img.shields.io/badge/T1040-Cleartext%20Creds-red)
![T1048](https://img.shields.io/badge/T1048-Exfiltration-purple)

---

## What this lab proves

For every scenario, the same three questions get answered from the capture alone. Who is the aggressor. What is the technique and did it succeed. What do I escalate with. Each attack is captured the way a real sensor or endpoint sees it, analysed packet by packet, then mapped to a detection. The four scenarios cover reconnaissance, credential attack, credential exposure, and data exfiltration, the core of what lands in a Tier 1 queue.

---

## Lab topology

| Host | Role | IP |
|---|---|---|
| Kali Linux | Attacker | 192.168.64.15 |
| Ubuntu Server (wazuh-manager) | Target and Linux telemetry | 192.168.64.12 |
| Windows 11 (JAMES-VM) | Target and capture host | 192.168.64.17 |
| macOS host | Splunk indexer and Wazuh manager | n/a |

Capture tooling is Wireshark, tshark, and tcpdump, all captured victim side.

---

## Scenarios and findings

| # | Scenario | Technique | What the packets proved | Detection tie in |
|---|---|---|---|---|
| 01 | Port scan | T1046 | Read three open ports (135, 139, and 445) from raw SYN ACK replies, matching nmap ground truth with no scanner output. One source sweeping 1000 distinct ports. | Suricata sid 1000001 |
| 02 | SSH brute force | T1110 | Confirmed the brute force from repeated encrypted SSH negotiations. Packets flag the attack, the auth log confirms the compromise. | Suricata sid 1000002, Splunk linux_secure |
| 03 | Cleartext credentials | T1040 | Recovered a live username and password straight from the packets, no decryption. An exposure finding rather than an attack. | Cleartext protocol egress alerting |
| 04 | DNS exfiltration | T1048 | Decoded the exact exfiltrated data back out of DNS query subdomains, recovering SSN, card number, and password. | Sysmon Event ID 22, DNS query analytics |

---

## The through line

One idea runs across all four scenarios. Packets tell you different things depending on the protocol, and knowing which source to trust for which answer is the actual skill.

A port scan is fully readable, the open ports are right there in the SYN ACK replies. SSH is encrypted, so the packets flag the brute force but the auth log confirms the breach. Cleartext FTP hides nothing, the password sits in the query in plain text. DNS exfil hides the data in plain sight inside query names, and decoding it proves exactly what left. Packets flag, logs confirm, and the analyst knows the difference.

---

## Repository contents

| Path | Purpose |
|---|---|
| [SETUP.md](SETUP.md) | Capture environment build, Wireshark, tshark, and Npcap on Windows |
| [analysis/01-portscan.md](analysis/01-portscan.md) | Port scan analysis, T1046 |
| [analysis/02-ssh-bruteforce.md](analysis/02-ssh-bruteforce.md) | SSH brute force analysis, T1110 |
| [analysis/03-cleartext-creds.md](analysis/03-cleartext-creds.md) | Cleartext credential exposure, T1040 |
| [analysis/04-dns-exfil.md](analysis/04-dns-exfil.md) | DNS exfiltration analysis, T1048 |
| [docs/TIER1-PLAYBOOK.md](docs/TIER1-PLAYBOOK.md) | Packet analysis scenario strategy |
| [docs/FILTERS.md](docs/FILTERS.md) | Tier 1 Wireshark and tshark filter kit |
| [docs/SPLUNK-TIER1-PLAYBOOK.md](docs/SPLUNK-TIER1-PLAYBOOK.md) | Splunk triage mental model |
| [docs/TIER1-DRILLS.md](docs/TIER1-DRILLS.md) | Pattern recognition practice program |
| [docs/TIER1-DAILY-PRACTICE.md](docs/TIER1-DAILY-PRACTICE.md) | Weekly drill rotation, verified against live data |
| pcaps/ | Raw captures for each scenario |

---

## Method notes

Every capture is scoped at capture time with a BPF filter to the two hosts involved, so each PCAP is a clean, self contained artifact with no unrelated noise. Findings are stated honestly, including limits. The SSH scenario openly notes that packets cannot prove the compromise on their own and require log correlation. Provenance for each capture is recorded with capinfos.

---

## Status

Complete. All four scenarios captured, analysed, and documented, with the supporting Tier 1 playbooks and a live data verified drill rotation.
