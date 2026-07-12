# Splunk Tier 1 Triage Playbook & Mental Model

The strategic reference for **what a Tier 1 SOC analyst is actually doing in Splunk**,
and how to think about an alert rather than guessing. The skill is not reading every
log it's knowing what to search, what to ignore, and when to escalate.

---

## The core reality of the job

In Splunk you are looking at **all logs, from all hosts, all the time.** You are not
handed a single neat incident you are triaging a **queue** of alerts, and often
searching to *find* the incident in the first place.

The mistake that sinks juniors is trying to read everything. You can't. The job is to
**filter a firehose**: narrow fast, aggregate into a picture, decide, move on. Volume
is the enemy; aggregation is the weapon.

---

## The Tier 1 triage funnel (the mental model)

Every alert goes through the same funnel. This is the shape to burn in:

```
   ALERT FIRES
       │
       ▼
1. TRIAGE      → Is this real, or a known false positive?
       │
       ▼
2. SCOPE       → Who / what / when? Which host, user, source IP, time window?
       │
       ▼
3. INVESTIGATE → What actually happened? Pivot across logs to build the story.
       │
       ▼
4. DECIDE      → Benign / false positive / true positive → escalate?
       │
       ▼
5. DOCUMENT    → IOCs, timeline, verdict, escalation notes for Tier 2.
```

You are always somewhere in this funnel. If you feel lost, ask which stage you're in.

---

## The five triage questions

Every alert, answer these in order:

1. **Is it real?** Does the alert reflect actual malicious/anomalous activity, or a
   known benign pattern (a vuln scanner you own, a service account, patch activity)?
2. **What's the scope?** Which host(s), user(s), source IP(s), and time window?
3. **What happened?** Reconstruct the sequence by pivoting across sourcetypes.
4. **How bad is it?** One failed login vs 500 then a success. Recon vs active C2.
5. **What do I do?** Close as false positive, or escalate to Tier 2 with IOCs and a
   timeline.

---

## The core triage skills (what you actually do at the keyboard)

- **Narrow first, always.** Start every search with `index=`, `sourcetype=`, and a
  bounded **time range**. Never run a bare all-time search you'll drown and time out.
- **Aggregate before you read.** Use `stats count by ...` to see the *shape* of
  activity before reading individual events. Counts reveal patterns; raw logs hide
  them.
- **Pivot on a key field.** Once you have a suspect host/IP/user, pivot: search that
  entity across *other* sourcetypes to build the full story
  (auth logs → process logs → network logs).
- **Compare to baseline.** "Is 88 failed logins abnormal?" is only answerable against
  what normal looks like for that host. Tier 1 develops a sense of baseline over time.
- **Know your false positives.** Half of Tier 1 is recognising benign patterns fast so
  you can dismiss them and spend your time on what's real.

---

## The scenarios (what fires, and how to triage it)

The exact SPL lives in the SPL cheat-sheet doc. This is the strategy for each.

### 1. Port Scan / Recon T1046
- **Fires from:** firewall / IDS logs, or connection logs showing one source hitting
  many ports.
- **Triage:** one `src_ip` reaching many distinct `dest_port` values on one `dest_ip`
  in a short window. Aggregate with `stats dc(dest_port) by src_ip`.
- **Verdict question:** an owned scanner, or an adversary? Usually the first touch in
  an attack chain.

### 2. Brute-Force Authentication T1110
- **Fires from:** Windows Security logs (EventCode 4625 failed logon), Linux auth
  logs, RDP/SSH.
- **Triage:** count failures per source/user, then check for a **success (4624)**
  after the burst. `stats count by src_ip, user`, then look for the one that flips from
  fail to success.
- **Verdict question:** *did they get in?* A success after many failures is the
  compromise escalate immediately.

### 3. Suspicious / Anomalous Authentication T1078
- **Fires from:** auth anomalies, off-hours logins, logins from unusual sources,
  impossible-travel patterns.
- **Triage:** who authenticated, from where, when, over what. Correlate against
  expected behaviour for that account.
- **Verdict question:** legitimate, misconfiguration, or account misuse?

### 4. C2 / Beaconing / Exfil T1071, T1048, T1041
- **Fires from:** proxy / DNS / firewall logs showing a host talking to a suspicious
  external destination.
- **Triage:** regular intervals (beacon regularity), unusual domains, large or odd
  outbound volume. `stats count by dest_ip, dest_port` and bucket by time to spot
  regular beaconing.
- **Verdict question:** is this host phoning home or leaking data?

### 5. Process / Endpoint Anomaly T1059, T1204
- **Fires from:** Sysmon (EventCode 1 process creation, 4104 script block logging).
- **Triage:** suspicious parent-child chains, encoded/obfuscated PowerShell, unusual
  binaries in unusual paths.
- **Verdict question:** normal admin activity or malicious execution?

---

## Beyond this lab: the wider Tier 1 alert landscape

The five scenarios above are the **network, authentication, and endpoint attack
fundamentals** the techniques that underpin most incidents, and the ones this lab
demonstrates with real artifacts. A live SOC queue is broader. The mix depends
entirely on the org's tooling (a cloud-identity-heavy shop sees mostly the last few;
an EDR-heavy shop sees mostly malware alerts). These are listed for landscape
awareness the recurring categories a Tier 1 should recognise, not detections built
in this lab:

| # | Alert type | Typically fires from | Core triage question |
|---|---|---|---|
| 6 | Malware / AV / EDR detection | Defender, CrowdStrike, SentinelOne | Did it execute? Quarantined? Host clean now? |
| 7 | Phishing / email security | Proofpoint, Mimecast, Defender for O365, user reports | Who clicked? What's the payload? How many targeted? |
| 8 | Data Loss Prevention (DLP) | DLP tooling | What data, where did it go, intentional? |
| 9 | Privilege escalation / account change | Windows Security (4720, 4728/4732, 4672) | Was this authorized/ticketed? |
| 10 | Malicious / suspicious web traffic | Proxy logs | Known-bad or newly-registered domain? |
| 11 | Lateral movement | Network + Windows logon types (4624 Type 3) | One host reaching many others internally? |
| 12 | Impossible travel / geo-anomaly | Azure AD, Okta, cloud identity | Same account, two locations, short window? |
| 13 | Persistence mechanisms | Sysmon (EID 12/13, EID 1), Windows 7045 | New task/service/run-key authorized? |
| 14 | System health / log-source down | Splunk internal, forwarder status | Operational, or an attacker covering tracks? |

Two of these deserve a note. **Impossible travel** is increasingly the highest-volume
real-world alert as identity moves to the cloud. **Log-source-down** looks purely
operational but is sometimes an attacker disabling logging never dismiss it
reflexively.

---

## The escalation decision

Tier 1 closes or escalates. When you escalate to Tier 2, hand over:

- **Verdict** true positive / suspicious / needs deeper analysis.
- **Scope** affected host(s), user(s), IP(s).
- **Timeline** first-seen to last-seen, key events in order.
- **IOCs** IPs, domains, hashes, usernames, ports.
- **What you checked** so Tier 2 doesn't repeat your work.

A clean escalation is itself a Tier 1 skill. Vague "this looks bad" tickets get bounced.

---

## Working principle

> The logs are the truth, but they're **scattered**. Your job is to gather the right
> ones fast, aggregate them into a picture, and decide.

**Narrow → aggregate → pivot → decide → document.** That is Tier 1 in Splunk.
