---
layout: post
title: "SSH Infrastructure Hardening and SOC Dashboard Deployment"
date: 2026-02-19
categories: [homelab, siem, blue-team, infrastructure]
tags: [splunk, ssh, credential-hygiene, dashboard, spl, windows-events, active-directory]
excerpt: "Credential hygiene remediation and initial SIEM monitoring capability for a Windows Active Directory security lab."
---

Credential hygiene remediation and initial SIEM monitoring capability for a Windows Active Directory security lab.

<!--more-->

## Overview

This session addressed two operational priorities: hardening the SSH automation pipeline for a remote Ubuntu server running Splunk Enterprise, and deploying the first security monitoring dashboard in the SIEM. The work closed out credential security findings from a previous Defender false positive investigation and established baseline authentication monitoring for the lab's Active Directory domain.

## SSH Authentication Hardening

Three Python automation scripts managing the Splunk server were identified as using a password-piping pattern for privileged command execution. The `sudo -S` flag, combined with `echo` to pass credentials through stdin, presented both a security concern (credentials in command strings) and an operational issue (triggering endpoint detection heuristics).

### Remediation Steps

**Key-based authentication** replaced password-based SSH login:

```bash
ssh-keygen -t ed25519 -C "homelab-automation"
scp $HOME\.ssh\id_ed25519.pub flux@192.168.1.186:~/pubkey.tmp
```

The public key was deployed to the server's `authorized_keys` with restrictive permissions (`chmod 700 ~/.ssh`, `chmod 600 ~/.ssh/authorized_keys`).

**Passwordless sudo** replaced the `echo '{password}' | sudo -S` pattern:

```bash
# /etc/sudoers.d/flux-nopasswd
flux ALL=(ALL) NOPASSWD: ALL
```

This eliminated all password dependencies from the automation pipeline. The three scripts (`ssh_diag.py`, `ssh_fix_secondbrain.py`, `ssh_fix2.py`) were refactored to use key-based authentication with paramiko's default key discovery, and `sudo` commands no longer require credential piping.

### Endpoint Detection Resolution

The Windows Defender `Trojan:Win32/Leonem!rfn` alerts from the previous session were confirmed resolved (`ActionSuccess: True` across all detections). A targeted folder exclusion was applied to the scripts directory following the principle of least privilege -- scoped to the specific automation scripts folder rather than a broader project or user-level exclusion.

## SOC Overview Dashboard

A four-panel Classic Dashboard was deployed in Splunk Enterprise, providing baseline security monitoring for the lab's Active Directory environment.

### Data Inventory

The Splunk instance contained 30,909 events across three Windows Event Log sourcetypes:

| Sourcetype | Event Count |
|------------|-------------|
| WinEventLog:Security | 28,577 |
| WinEventLog:System | 1,877 |
| WinEventLog:Application | 455 |

### Dashboard Panels

**Panel 1: Failed Logins (Event ID 4625)**

```
index=main sourcetype="WinEventLog:Security" EventCode=4625
| stats count by Account_Name, Source_Network_Address
| sort -count
```

Results showed two failed authentication attempts, both originating from the VirtualBox host adapter (192.168.56.1). This establishes a clean baseline -- any significant increase in failed logon volume would indicate brute force or credential stuffing activity.

**Panel 2: Successful Logins (Event ID 4624)**

```
index=main sourcetype="WinEventLog:Security" EventCode=4624 Logon_Type!=3 Logon_Type!=5
| stats count by Account_Name, Logon_Type, Source_Network_Address
| sort -count
```

Network (Type 3) and Service (Type 5) logons were excluded to surface interactive human authentication events. Machine accounts (identifiable by the `$` suffix) dominate the results, with the `Administrator` account as the only human interactive logon.

**Panel 3: Privileged Activity (Event ID 4672)**

```
index=main sourcetype="WinEventLog:Security" EventCode=4672
| stats count by Account_Name
| sort -count
| where NOT like(Account_Name, "%$")
```

Machine accounts were filtered using the `where NOT like` clause. The `Administrator` account showed 74 privilege assignment events across historical sessions -- consistent with expected domain admin usage patterns.

**Panel 4: Login Activity Over Time**

```
index=main sourcetype="WinEventLog:Security" (EventCode=4624 OR EventCode=4625)
| timechart span=1h count by EventCode
```

The timeline visualization reveals a characteristic burst pattern correlating with lab session activity. Extended flat periods indicate VMs were offline. This panel enables baseline pattern recognition -- anomalous activity during known-quiet periods would be immediately visible.

## Operational Value

The SSH hardening eliminates credential exposure in the automation pipeline, resolving both the security concern and the endpoint detection false positives. The SOC Overview dashboard provides continuous monitoring capability for the three most critical authentication event types in a Windows Active Directory environment. Together, these changes reduce the lab's attack surface while simultaneously improving detection visibility.
