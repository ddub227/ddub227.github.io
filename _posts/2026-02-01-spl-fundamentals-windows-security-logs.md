---
layout: post
title: "SPL Fundamentals: Windows Security Log Analysis and Kerberos Authentication"
date: 2026-02-01
categories: [homelab, siem, blue-team]
tags: [splunk, spl, windows-security, kerberos, log-analysis, soc]
excerpt: "Hands-on SPL practice session analyzing Windows Security events, investigating unknown accounts with pivot techniques, visualizing logon patterns, and understanding Kerberos authentication flow."
---

First comprehensive SPL practice session in the homelab. This post covers querying Windows Security events, investigating unknown accounts using pivot techniques, analyzing admin activity patterns, and examining Kerberos authentication flow.

<!--more-->

---

## Session Overview

| Field | Value |
|-------|-------|
| **Focus** | SIEM log analysis, SPL query development, Windows Security Event interpretation, Kerberos authentication flow |
| **Systems Used** | Splunk Enterprise, DC01 (Windows Server 2022), Home Server (Ubuntu/Docker) |
| **Goal** | SPL query practice and log analysis fundamentals |

---

## Key Accomplishments

- **Baseline establishment:** Queried 28,661 security events to identify top event codes and normal activity patterns
- **Account investigation:** Used pivot technique to investigate unknown machine accounts - closed as benign legacy hostnames
- **Admin activity review:** Analyzed Event 4672 to confirm minimal privileged account usage
- **Time-series analysis:** Visualized logon patterns with `timechart` to understand activity baselines
- **Kerberos deep-dive:** Examined 4768/4769 events to understand TGT and Service Ticket authentication flow
- **Infrastructure fix:** Diagnosed and permanently resolved recurring WiFi disconnects on home server

---

## Exercise 1: Baseline Query

Establish what "normal" looks like by identifying the most common event codes:

```spl
index=main host=DC01 sourcetype="WinEventLog:Security" | stats count by EventCode | sort -count | head 10
```

**Findings:** Top event codes on DC01:

| Event Code | Count | Meaning |
|------------|-------|---------|
| 4624 | 8,593 | Successful logon |
| 4672 | 7,681 | Special privileges assigned |
| 4634 | 7,677 | Logoff |
| 5379 | 2,447 | Credential Manager read |
| 4907 | 672 | Audit policy changed |
| 4769 | 363 | Kerberos service ticket |
| 4768 | 169 | Kerberos TGT request |
| 4799 | 165 | Group membership enumerated |

**Total events:** 28,661

---

## Exercise 2: Account Activity Analysis

Identify which accounts are logging into the domain controller:

```spl
index=main host=DC01 EventCode=4624 | stats count by Account_Name | sort -count | head 10
```

**Findings:**

| Account | Count | Type |
|---------|-------|------|
| DC01$ | 7,028 | Machine account |
| SYSTEM | 728 | Local system |
| WS01$ | 446 | Workstation machine account |
| tuser | 409 | Test user |
| WIN-9A6EKCLN088$ | 76 | Unknown (investigated) |
| Administrator | 48 | Domain admin |
| MINWINPC$ | 29 | Unknown (investigated) |

The unknown accounts (`WIN-9A6EKCLN088$` and `MINWINPC$`) triggered an investigation.

---

## Exercise 3: Unknown Account Investigation

Using the pivot technique to investigate suspicious accounts:

**Pivot 1 - Logon Types:**
```spl
index=main host=DC01 Account_Name="WIN-9A6EKCLN088$" | stats count by EventCode, Logon_Type | sort -count
```
Result: Type 5 (service) and Type 2 (interactive) - local activity only

**Pivot 2 - Source IP:**
```spl
index=main host=DC01 Account_Name="WIN-9A6EKCLN088$" EventCode=4624 | stats count by Source_Network_Address | sort -count
```
Result: 127.0.0.1 and blank (local only)

**Conclusion:** Both accounts are old computer names from before VMs were renamed. Benign - closed as false positive.

This exercise demonstrated the SOC investigation methodology:
1. Find anomaly (unknown accounts)
2. Pivot to understand context (logon types)
3. Pivot again for more detail (source IPs)
4. Correlate and conclude
5. Document findings

---

## Exercise 4: Failed Logon Analysis

Check for brute force indicators:

```spl
index=main host=DC01 EventCode=4625 | stats count by Account_Name, Failure_Reason | sort -count
```

**Findings:** Only 8 failed logons total - healthy quiet lab with no brute force indicators.

---

## Exercise 5: Admin Activity Analysis

Monitor privileged account usage:

```spl
index=main host=DC01 EventCode=4672 | stats count by Account_Name | sort -count | head 10
```

**Key insight:** Administrator has only 4 privileged logons - excellent security posture. Most 4672 events come from machine accounts (DC01$) doing their normal work.

---

## Exercise 6: Time-Based Analysis

Visualize when activity happens to spot anomalies:

```spl
index=main host=DC01 EventCode=4624 | timechart span=1h count
```

![Splunk Timechart Visualization](/assets/images/2026-02/splunk-timechart.png)

**Findings:**
- High activity (200-280/hour) during active lab periods
- Near-zero activity during off-hours
- Gap correlates with server offline periods

**Investigation:** DC01 showed 273 logons during 3-4 AM. Drilled down and confirmed all were machine account (DC01$) background activity, not human accounts. Domain controllers generate constant service authentication even when idle - this is normal.

---

## Exercise 7: Kerberos Authentication Review

Understand authentication flow through Kerberos events:

```spl
index=main host=DC01 EventCode IN (4768, 4769) | stats count by EventCode, Account_Name | sort -count | head 15
```

**Findings:**

| EventCode | Account | Count | Meaning |
|-----------|---------|-------|---------|
| 4769 | DC01$@LAB.LOCAL | 6 | Machine service tickets |
| 4768 | DC01$ | 3 | Machine TGT requests |
| 4768 | Administrator | 2 | Admin TGT requests |
| 4769 | Administrator@LAB.LOCAL | 2 | Admin service tickets |

**Key insight:** The `@LAB.LOCAL` suffix is the Kerberos realm (same as AD domain name). Accounts appear with different formats because 4768 logs simple names while 4769 logs full principal names.

**Kerberos flow observed:**
1. Administrator logged in → TGT issued (4768)
2. Administrator accessed resource → Service Ticket issued (4769)

---

## Concepts Learned

### Windows Event Codes (Security Log)

| Code | Meaning | SOC Relevance |
|------|---------|---------------|
| 4624 | Successful logon | Track who's accessing systems |
| 4625 | Failed logon | Detect brute force attempts |
| 4634 | Logoff | Pair with 4624 for session tracking |
| 4672 | Special privileges assigned | Admin activity monitoring |
| 4768 | Kerberos TGT request | Initial authentication |
| 4769 | Kerberos service ticket | Service access |

### Logon Types

| Type | Name | Meaning |
|------|------|---------|
| 2 | Interactive | Physical/console logon |
| 3 | Network | Remote resource access |
| 5 | Service | Service starting |
| 10 | RemoteInteractive | RDP session |

### Machine Accounts

- End with `$` symbol (e.g., DC01$, WS01$)
- Represent computers, not humans
- Generate constant background authentication traffic
- High volume from machine accounts at 3 AM = normal
- High volume from human accounts at 3 AM = suspicious

---

## Troubleshooting: WiFi Power Save

The home server (running Splunk in Docker) kept disconnecting. After three SSH timeouts in one session, I diagnosed the root cause.

**Diagnosis:**
```bash
ip link show          # Found wlp2s0 in DORMANT mode
iw dev wlp2s0 get power_save  # "Power save: on"
```

**Root Cause:** Linux WiFi power management putting adapter to sleep (laptop repurposed as server)

**Permanent Fix:**
```bash
# Disable power save immediately
sudo iw dev wlp2s0 set power_save off

# Create systemd service for persistence
sudo tee /etc/systemd/system/wifi-powersave-off.service << 'EOF'
[Unit]
Description=Disable WiFi Power Save
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/iw dev wlp2s0 set power_save off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable wifi-powersave-off.service
```

**Lesson:** Laptops repurposed as servers need power management disabled.

---

## Skills Practiced

| Skill | Activity | SOC Relevance |
|-------|----------|---------------|
| SPL Queries | stats, sort, head, timechart commands | Core SIEM skill |
| Log Analysis | Windows Security event interpretation | Daily SOC work |
| Investigation | Pivot technique for unknown accounts | Alert triage |
| Time-Series Analysis | Visualizing logon patterns over time | Anomaly detection |
| Kerberos Analysis | Interpreting 4768/4769 authentication events | Detecting lateral movement |
| Linux Administration | systemd service creation, wireless config | Server management |

---

## Key Takeaway

> Understanding "normal" is the foundation of detecting "abnormal." A domain controller generating 200+ machine account logons at 3 AM is expected behavior - the same activity from a human account would be a red flag. Baselining your environment through hands-on SPL practice builds the intuition needed to spot real threats in a sea of legitimate noise.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Splunk Enterprise | SIEM platform for all SPL queries and log analysis |
| Splunk Universal Forwarder | Ships Windows logs from DC01 to Splunk |
| Docker | Hosts Splunk container on home server |
| SSH | Remote management of home server |
| PowerShell | Network diagnostics from Windows host |
| Windows Server 2022 | Domain Controller generating security events |
| iw | Linux wireless diagnostics for troubleshooting |
| systemd | Created persistent service to disable WiFi power save |
