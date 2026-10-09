---
layout: post
title: "Detection Engineering Pipeline: Sigma Rules, SIEM Alert Deployment, and Disaster Recovery"
date: 2026-02-21
categories: [homelab, siem, blue-team]
tags: [splunk, sigma, detection-engineering, mitre-attack, spl, nist-csf, backup, disaster-recovery, pysigma]
excerpt: "Detection rule development using the Sigma format, SIEM timestamp remediation, MITRE-mapped alert deployment, and infrastructure backup planning for a Windows Active Directory security lab."
---

Detection rule development using the Sigma format, SIEM timestamp remediation, MITRE-mapped alert deployment, and infrastructure backup planning for a Windows Active Directory security lab.

<!--more-->

## Overview

This report covers four workstreams completed against the lab.local Active Directory environment: diagnosing and remediating a SIEM timestamp desynchronization that broke event correlation, authoring and validating three Sigma detection rules with pySigma conversion to Splunk SPL, deploying MITRE ATT&CK-mapped alerts for credential access and valid account abuse, and closing the final two NIST CSF 2.0 infrastructure gaps with backup and recovery planning.

The Sigma rules are published as a public repository: [ddub227/sigma-detection-rules](https://github.com/ddub227/sigma-detection-rules).

## SIEM Timestamp Remediation

### Problem

Splunk events displayed an 8-hour discrepancy between the `_time` column and the raw event data. An event that occurred at approximately 11:00 AM local time appeared as 3:50 PM in the Splunk Time column and 7:50 AM in the raw event body.

### Root Cause Analysis

Three systems were operating in three different timezones:

| System | Timezone | UTC Offset |
|--------|----------|------------|
| Host machine (Windows 11) | Eastern | UTC-5 |
| DC01 (Windows Server 2022) | Pacific | UTC-8 |
| Splunk (Ubuntu/Docker) | UTC | UTC+0 |

The Windows Event Log service embeds timestamps in local time without timezone metadata. Splunk, operating in UTC, ingested these Pacific-time timestamps and interpreted them as UTC. The 8-hour gap is the exact offset between Pacific and UTC.

This is a common failure mode in multi-source SIEM environments. Any log source that writes local time without a timezone qualifier will produce correlation errors when the SIEM assumes UTC.

### Resolution

DC01's system timezone was changed to UTC via Settings > Time & Language. However, a settings change alone was insufficient. The Event Log service caches the timezone offset at startup and does not re-read it dynamically. Post-change verification with `Get-WinEvent` still showed the old offset until a full VM restart flushed the cached value.

After reboot, Splunk's Time column and raw event timestamps matched. All log sources in the environment now operate in UTC.

### Operational Implication

Timezone drift across log sources silently corrupts alert correlation and investigation timelines. An alert triggering on a 5-minute aggregation window becomes unreliable when the underlying events are offset by hours. UTC standardization across all log sources is a prerequisite for accurate detection, not an optimization.

## Sigma Detection Rules

Three detection rules were authored in the Sigma format, validated through the pySigma converter using the `splunk_windows` processing pipeline, and tested against live attack data in Splunk. Each rule maps to a MITRE ATT&CK technique.

### Rule 1: Kerberoasting - RC4 TGS Request (T1558.003)

Detection logic: Event ID 4769 (Kerberos TGS request) where `TicketEncryptionType` equals `0x17` (RC4), excluding machine accounts (identified by the `$` suffix on `ServiceName`).

```yaml
detection:
    selection:
        EventID: 4769
        TicketEncryptionType: '0x17'
    filter_machine_accounts:
        ServiceName|endswith: '$'
    condition: selection and not filter_machine_accounts
```

Rationale: Modern Active Directory environments default to AES encryption (0x11/0x12) for Kerberos tickets. Kerberoasting tools such as impacket-GetUserSPNs request RC4 (0x17) because RC4-encrypted TGS-REP hashes are crackable offline with hashcat. The machine account filter eliminates noise from legitimate AD computer account operations that routinely generate RC4 TGS requests.

Converted SPL:

```
index=main EventCode=4769 Ticket_Encryption_Type=0x17
| where NOT match(Service_Name, "\$")
| table _time, Account_Name, Service_Name, Client_Address
| sort -_time
```

Validation: Attack simulation with `impacket-GetUserSPNs` against the `svc_sqldb` service account confirmed 4769 events with `0x17` encryption. The rule returned only attack-originated events after machine account filtering.

### Rule 2: Brute Force - Failed Logon Attempts (T1110)

Detection logic: Event ID 4625 (failed logon) with a SIEM-side threshold of 5 or more events from the same source IP within a 5-minute window.

```yaml
detection:
    selection:
        EventID: 4625
    condition: selection
```

Implementation note: Sigma's correlation syntax supports `count() by IpAddress > 5` with a `timeframe: 5m` parameter, but pySigma converters require a separate correlation file. The threshold logic is applied at the SIEM layer:

```
source="WinEventLog:Security" EventCode=4625
| bin _time span=5m
| stats count by Source_Network_Address, _time
| where count > 5
```

Rationale: Individual 4625 events are common (mistyped passwords, locked accounts). The value is in the aggregation pattern: sustained failures from a single source IP indicate automated credential attacks rather than human error.

### Rule 3: After Hours Privileged Logons (T1078)

Detection logic: Event ID 4672 (special privileges assigned to new logon) excluding the SYSTEM account and machine accounts, combined with SIEM-side time-of-day filtering for events outside 08:00 to 18:00.

```yaml
detection:
    selection:
        EventID: 4672
    filter_system:
        SubjectUserName: 'SYSTEM'
    filter_machine_accounts:
        SubjectUserName|endswith: '$'
    condition: selection and not filter_system and not filter_machine_accounts
```

SIEM-side time filter:

```
| where date_hour < 8 OR date_hour >= 18
```

Rationale: Sigma does not natively support time-of-day conditions, so the temporal component is enforced at the SIEM layer. Privileged logons during off-hours are a high-fidelity indicator of compromised credentials or unauthorized access. The SYSTEM and machine account filters prevent the rule from firing on routine Windows background operations.

### Conversion Pipeline

All three rules were converted from Sigma YAML to Splunk SPL using the standard toolchain:

```bash
pip install pySigma pySigma-backend-splunk sigma-cli
sigma convert -t splunk -p splunk_windows rules/windows/credential_access/kerberoasting_rc4_tgs_request.yml
```

The converted SPL was compared against the hand-written detection queries already running in Splunk. Field name mappings (e.g., `TicketEncryptionType` in Sigma to `Ticket_Encryption_Type` in Splunk) are handled by the `splunk_windows` processing pipeline.

## Alert Deployment

Three alerts are now operational in Splunk, all running on 15-minute schedules:

| Alert | MITRE Technique | Event ID | Severity | Schedule |
|-------|----------------|----------|----------|----------|
| Kerberoasting Detection - RC4 TGS Request | T1558.003 | 4769 | High | Every 15 min |
| Brute Force Detection - Failed Logons | T1110 | 4625 | Medium | Every 15 min |
| After Hours Privileged Logon | T1078 | 4672 | High | Every 15 min |

Coverage spans two MITRE ATT&CK tactics:

**Credential Access:** Kerberoasting and brute force detection address the two most common credential theft vectors in Active Directory environments.

**Valid Accounts (Defense Evasion / Persistence):** After-hours privileged logon detection targets post-compromise activity where stolen credentials are used outside normal operating patterns.

The Kerberoasting alert was validated by re-executing the attack simulation and confirming triggers appeared in Splunk's Triggered Alerts view. Alert time ranges match the cron interval (last 15 minutes) to prevent re-alerting on historical events.

## SOC Dashboard Update

The SOC Overview dashboard was expanded to five panels:

| Panel | Event ID | Purpose |
|-------|----------|---------|
| Failed Logins | 4625 | Brute force / credential stuffing baseline |
| Successful Logins | 4624 | Interactive authentication tracking |
| Privileged Activity | 4672 | Privilege escalation monitoring |
| Login Activity Over Time | 4624, 4625 | Temporal pattern recognition |
| Kerberoasting - RC4 TGS Requests | 4769 | RC4 ticket request detection |

The new Kerberoasting panel surfaces RC4 TGS requests in a statistics table with `_time`, `Account_Name`, `Service_Name`, and `Client_Address` columns, providing immediate visibility into potential Kerberoasting activity without requiring an analyst to run ad-hoc queries.

## NIST CSF 2.0 Gap Closure

A prior self-assessment against NIST CSF 2.0 identified two categories rated "Not Implemented":

**PR.IR (Infrastructure Resilience):** No backup capability existed. A single hardware failure or VM corruption would require a full environment rebuild.

**RC.RP (Recovery Planning):** No documented recovery procedures. Response to a failure event would be entirely improvised.

### Backup Infrastructure

**VirtualBox Snapshot:** DC01_Baseline_2026-02-21 provides local rollback capability for configuration errors and failed updates.

**OVA Export:** DC01 exported as a portable appliance (10.6 GB compressed from a 60 GB allocated disk). OVA files are VirtualBox-agnostic and can be imported on any machine running a compatible hypervisor.

**3-2-1 Strategy:** 3 copies (original VM + local OVA + cloud OVA), 2 media types (local disk + Google Drive), 1 offsite (Google Drive).

### Recovery Plan

A full recovery plan was documented with four scenarios and target recovery times:

| Scenario | Recovery Source | RTO |
|----------|---------------|-----|
| VM corruption / misconfiguration | VirtualBox snapshot | ~5 minutes |
| Accidental VM deletion | Local OVA import | ~15-30 minutes |
| Host hard drive failure | Google Drive OVA download + import | 1-2 hours |
| Splunk server failure | Docker redeploy + config rebuild | 1-3 hours |

Each scenario includes step-by-step procedures, network reconfiguration requirements, and a post-recovery verification checklist covering AD services, DNS resolution, Universal Forwarder status, and alert functionality.

Both PR.IR and RC.RP moved from "Not Implemented" to "Partial." Full implementation requires completing the automated weekly export schedule and validating recovery by executing each scenario.

## Summary

| Workstream | Outcome |
|------------|---------|
| Timestamp remediation | DC01 standardized to UTC; Event Log cache behavior documented |
| Sigma rules | 3 rules authored, converted, validated, and published to GitHub |
| Alert deployment | 3 MITRE-mapped alerts operational on 15-minute schedules |
| Dashboard | SOC Overview expanded to 5 panels with Kerberoasting visibility |
| NIST CSF gaps | PR.IR and RC.RP closed with backup infrastructure and recovery plan |

Detection engineering is a pipeline with discrete failure points at every stage. A Sigma rule that does not convert cleanly to SPL is unusable. SPL that converts but does not match live event field names returns nothing. An alert that fires but has no triage procedure generates noise, not signal. Each stage requires independent validation against real data before the rule provides operational value.
