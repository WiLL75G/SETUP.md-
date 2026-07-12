# SSH Brute Force Analysis (T1110)

Reconstructing an SSH brute-force attack from a raw packet capture, and confirming the
compromise by correlating packets with the auth log. The central lesson: because SSH is
encrypted, packets flag the attack but cannot prove the breach on their own.

| Field | Value |
|---|---|
| Technique | T1110 Brute Force |
| Attacker | Kali Linux, 192.168.64.15 |
| Target | Ubuntu Server (wazuh-manager), 192.168.64.12 |
| Capture | `ssh-bruteforce.pcap` 465 packets, 86 kB, ~74s window |
| Detection tie-in | Suricata sid 1000002 · Splunk `linux_secure` |

---

## Attack

Two hydra runs against SSH, using a deliberate two-account design so the capture
contains both outcomes:

```
# john password present in the wordlist → SUCCESS (the compromise)
hydra -l john -P /tmp/john-list.txt -t 4 ssh://192.168.64.12

# mary password absent → pure failure, never gets in
hydra -l mary -P /tmp/mary-list.txt -t 4 ssh://192.168.64.12
```

Ground truth: hydra reported a valid password found for john and none for mary.

## Capture setup

Captured victim-side on the Ubuntu box with tcpdump, scoped to the attacker↔victim SSH
conversation only:

```
sudo tcpdump -i enp0s1 -w ssh-bruteforce.pcap host 192.168.64.15 and host 192.168.64.12 and port 22
```

`capinfos` shows 465 packets at an average size of 170 bytes. That is roughly triple a
port scan's average SSH brute force is made of full encrypted connections
(handshake, key exchange, auth, teardown), not empty probes. Packet size alone
distinguishes "someone tried to log in" from "someone scanned me."

## What the packets revealed

**Connection attempts the SYNs, with timing:**
```
tshark -r ssh-bruteforce.pcap -Y "tcp.flags.syn==1 && tcp.flags.ack==0" -T fields -e frame.time_relative -e tcp.srcport
```
14 connections in three time bursts: one burst at t=0 and two at t≈47s onward. The ~47s
gap corresponds exactly to the launch of the second hydra run the attack timeline
reconstructs itself from the packets. Burst 1 is john's attack; the later bursts are
mary's. (Hydra pipelines multiple password attempts over each connection, so 14
connections carried all 51 attempts connection count is not attempt count.)

**Conversation breakdown size and duration:**
```
tshark -r ssh-bruteforce.pcap -q -z conv,tcp
```
Within john's burst, one connection (source port 33120) stands out: longest (~17s) and
heaviest (6,358 bytes). A successful authentication pushes slightly more exchange before
hydra closes the connection, so this is the prime suspect for the successful login.

**Protocol hierarchy:** unlike the port scan's empty upper layers, this capture shows a
real SSHv2 conversation protocol negotiation and key exchange repeated many times.
Repeated full SSH negotiations from one source is the brute force on the wire.

## The encryption wall and the correlation that solves it

Here the analysis hits a hard limit. SSH is encrypted: the password, the success, and
the failure all live inside the encrypted channel. The packets show one connection
behaving differently, which is a strong hint, but not proof of compromise.

Confirmation comes from correlating with the auth log in Splunk:

```spl
index=main host=wazuh-manager sourcetype=linux_secure "Accepted password"
| rex "for (?<user>\w+) from (?<src_ip>\d+\.\d+\.\d+\.\d+)"
| table _time, user, src_ip
```

Result: `Accepted password for john from 192.168.64.15`. mary, attacked identically,
produced only failed-password events. The log confirms in plain text what the packets
could only suggest.

## Detection takeaway

Repeated full SSH connections from a single source to port 22 in a short window, with
one connection deviating in duration and volume, indicate an SSH brute force (T1110)
with possible successful compromise.

**Packets flag the anomaly; logs confirm the compromise.** The PCAP identifies the
attack, the attacker, and the likely compromise window; the auth log proves the actual
breach. Neither source alone tells the whole story the correlation is the detection.

This maps to **Suricata sid 1000002** (connection-rate brute-force alert) and to the
Splunk `linux_secure` detection that distinguishes a failed brute force (mary) from a
successful one (john) by the presence of an `Accepted password` event after the burst.
