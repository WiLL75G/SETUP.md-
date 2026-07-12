# Cleartext Credential Exposure (T1040)

Recovering login credentials from a raw packet capture with no decryption and no cracking, because the protocol carried them in plaintext. Unlike a brute force or an exploit, this is usually a misconfiguration finding, which is a large share of real Tier 1 work.

| Field | Value |
|---|---|
| Technique | T1040 Network Sniffing |
| Server | Ubuntu (wazuh-manager), 192.168.64.12, FTP on port 21 |
| Client | Kali Linux, 192.168.64.15 |
| Capture | `cleartext-ftp.pcap`, 17 packets, ~2 kB |
| Credentials recovered | user `socuser`, password `Cleartext123` |

---

## Setup

A throwaway FTP server was stood up on the Ubuntu box using pyftpdlib, exposing a known test account so the recovered credentials could be verified against ground truth:

```
sudo python3 -m pyftpdlib -p 21 -u socuser -P Cleartext123 -d /tmp &
```

FTP is fully unencrypted. Every command, including the username and password, travels across the wire as plaintext. That is the point of the scenario.

## Capture setup

Captured victim side on the Ubuntu box with tcpdump, scoped to the client and server FTP conversation only:

```
sudo tcpdump -i enp0s1 -w cleartext-ftp.pcap host 192.168.64.15 and host 192.168.64.12 and port 21
```

The login was then performed from Kali. A single non interactive curl login was used to avoid the interactive client timing out:

```
curl ftp://192.168.64.12/ --user socuser:Cleartext123
```

## What the packets revealed

The credentials are readable directly in the packet list Info column, no reconstruction required:

```
Response: 220 pyftpdlib 1.5.9 ready
Request:  USER socuser
Response: 331 Username ok, send password
Request:  PASS Cleartext123
Response: 230 Login successful
```

Extracting just the credential commands with a display filter confirms it cleanly:

```
tshark -r cleartext-ftp.pcap -Y 'ftp.request.command == "USER" || ftp.request.command == "PASS"' -T fields -e ftp.request.command -e ftp.request.arg
```

Output:

```
USER    socuser
PASS    Cleartext123
```

Username and password recovered from raw packets, no decryption, no cracking. Following the TCP stream renders the entire login exchange as readable text, with the credentials in plain sight.

## Detection takeaway

This is not an attack signature. It is a credential exposure finding. A service was configured to accept authentication over an unencrypted protocol, so anyone positioned to capture traffic, a rogue device, a compromised host, or an insider, reads the credentials for free.

The detection logic is the presence of cleartext authentication protocols on the network at all, especially carrying credentials. The filter used above (`ftp.request.command == "USER"` or `"PASS"`) is itself the detection. The same idea extends to Telnet and HTTP Basic auth, where credentials are either plaintext or only Base64 encoded, which is encoding, not encryption.

The Tier 1 finding and recommendation: the FTP service on 192.168.64.12 accepts cleartext authentication, and the credentials for `socuser` were recovered from captured traffic. Recommend disabling FTP in favor of SFTP or FTPS, or restricting the service to a management network where the traffic cannot be sniffed by untrusted hosts.

## The reusable pattern

1. Identify the cleartext protocol in the capture, FTP, Telnet, or HTTP Basic.
2. Filter for the credential carrying commands or headers.
3. Read the username and password directly, or follow the TCP stream for the full exchange.
4. Treat it as an exposure finding, report the affected service and recommend an encrypted replacement.
