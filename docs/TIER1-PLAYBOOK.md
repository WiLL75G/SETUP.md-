# Tier 1 Packet Analysis — Scenario Playbook

A strategic reference for **what a Tier 1 SOC analyst is actually asked to do** when a
PCAP lands on their desk, and how to approach each scenario deliberately rather than
guessing. This sits above the filter cheat-sheet (`FILTERS.md`): the playbook tells
you *which investigation you're in*; the cheat-sheet tells you *which filters to run*.

---

## What actually triggers a Tier 1 packet analysis

Tier 1 rarely opens Wireshark on a whim. A PCAP lands on the desk because **something
upstream fired** — a SIEM alert, an IDS signature, a firewall log, an EDR detection —
and now the job is to **confirm or dismiss it**. So the real scenarios are "alert
types that come with a PCAP attached."

---

## The three questions (the mental template for every case)

Whatever the alert, Tier 1 answers the same three questions. Burn these in:

1. **Who is the aggressor, and who is the target?**
   (source/destination, internal vs external)
2. **What is the technique, and did it succeed?**
   (scan → any open? brute force → any success? C2 → confirmed beacon? exfil → did
   data actually leave?)
3. **What do I escalate with?**
   (IOCs: IPs, domains, hashes, URIs — the artifacts Tier 2 needs)

Everything you do in Wireshark is in service of answering those three.

---

## The scenarios

### 1. Port Scan / Reconnaissance — T1046

- **Alert trigger:** IDS "possible port sweep"; one host touching many ports.
- **Confirm by:** one source, many distinct destination ports, short window, high RST
  volume, minimal payload.
- **The verdict question:** Is this an owned vuln scanner, a benign tool, or a real
  adversary? Usually the **first real touch** in an attack chain.
- **Lab:** `01-portscan`.

### 2. Brute-Force Authentication — T1110

- **Alert trigger:** "Multiple failed logins" on SSH, RDP, SMB, or a web login.
- **Confirm by:** counting repeated auth attempts from one source to one service —
  and critically, looking for the **one success after many failures**.
- **The verdict question:** *Did they get in?* (Read from connection behavior when the
  protocol is encrypted — one long/large flow among many short ones = success.)
- **Lab:** `02-ssh-bruteforce`.

### 3. Cleartext Credential Exposure — T1040 / weak protocol use

- **Alert trigger:** audit or IDS flags HTTP, FTP, Telnet, or SNMP carrying auth.
- **Confirm by:** Follow Stream and read the credentials in plaintext.
- **The verdict question:** Are creds actually exposed? Often **not an attack** — a
  misconfiguration finding, which is a large share of real Tier 1 work.
- **Lab:** `03-cleartext-creds`.

### 4. C2 / Beaconing — T1071, T1571

- **Alert trigger:** EDR/SIEM flags a host talking to a suspicious external IP.
- **Confirm by:** regular beacon intervals, small consistent payloads, odd URIs,
  HTTP-over-443 evasion, DNS to strange domains.
- **The verdict question:** Is this malware phoning home? *The big one.*
- **Lab:** `04-dns-exfil`, plus the NetSupport RAT bonus PCAP.

### 5. Data Exfiltration — T1048, T1041

- **Alert trigger:** large or unusual outbound transfer.
- **Confirm by:** volume, destination, protocol — DNS TXT floods, big HTTPS uploads to
  non-corporate IPs, FTP-out.
- **The verdict question:** Did data actually leave, and where to? Often overlaps with
  C2.

### 6. Malware Delivery / Suspicious Download

- **Alert trigger:** proxy or EDR flags a download.
- **Confirm by:** **Export Objects** to pull the actual file out of the PCAP, then
  hash it for VirusTotal.
- **The verdict question:** Is the delivered file malicious?

---

## How this lab maps to the real job

The four project scenarios are not random — they rehearse the most common
alert-to-PCAP workflows:

| Lab scenario | Real-world scenario | Technique |
|---|---|---|
| `01-portscan` | 1 — Reconnaissance | T1046 |
| `02-ssh-bruteforce` | 2 — Brute force | T1110 |
| `03-cleartext-creds` | 3 — Cleartext exposure | T1040 |
| `04-dns-exfil` | 4 / 5 — C2 & exfiltration | T1048 / T1071 |
| Bonus (MTA.net PCAP) | 4 — C2 beaconing, cold | T1071 / T1219 |

This isn't a Wireshark tutorial — it's rehearsal of the four most common workflows a
Tier 1 analyst faces, plus a cold PCAP to prove the skill transfers.

---

## The one thing a lab can't fully teach: volume and noise

Lab PCAPs are clean — scoped to two hosts, no clutter. A **real** capture is 15,000+
packets of mostly-legitimate traffic where the malicious handful is buried. (A real
NetSupport RAT case meant filtering ~15,500 packets down to ~550 to isolate the C2.)

That's why the **bonus PCAP matters most**: it forces you to find signal in noise,
which is the actual Tier 1 challenge. Doing your own attack teaches the *pattern*;
doing a stranger's capture proves you *own* it.

---

## Working principle

> A SIEM alert gives you a **signal**. The PCAP gives you the **truth**.

Confirm or dismiss the alert. Answer the three questions. Produce IOCs. Escalate with
evidence. That is the job.
