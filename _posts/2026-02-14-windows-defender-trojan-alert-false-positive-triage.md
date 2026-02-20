---
layout: post
title: "Triaging a Trojan:Win32/Leonem Alert on a Homelab Workstation"
date: 2026-02-14
categories: [homelab, blue-team]
tags: [windows-defender, incident-response, false-positive, alert-triage, threat-intelligence, credential-hygiene, soc]
excerpt: "Windows Defender flagged custom SSH automation scripts as Trojan:Win32/Leonem!rfn. The alert was a false positive. The credential hygiene issue it exposed was not."
---

Windows Defender generated two Severe `Trojan:Win32/Leonem!rfn` alerts at 10:21 and 10:22 PM on February 14, 2026. Both were marked Active. The affected system was the primary workstation running Windows 11 Pro.

At the time, three Python scripts using Paramiko for SSH automation were being developed and tested against a homelab Ubuntu server. The last file save occurred at 10:12 PM -- nine minutes before the first alert, consistent with Defender's scan-on-write pipeline feeding files to the cloud ML engine for reputation analysis.

## Threat Profile

Leonem is a credential-stealing trojan that harvests stored browser and email passwords, captures keystrokes through DirectInput API hooks, and exfiltrates data via Discord webhooks. It detects sandboxes by querying `Win32_Bios` and `Win32_NetworkAdapter` through WMI. Persistence is achieved through Run registry keys and scheduled tasks. The malware attempts to disable Defender by writing `DisableAntiVirus` to the registry and can act as a dropper for secondary payloads.

Known IOCs for a Leonem infection include files in `C:\ProgramData\Microsoft\Network\Downloader\` mimicking BITS components, PowerShell processes probing for security products, outbound connections to Discord webhook endpoints and `ip-api.com`, and registry modifications targeting real-time protection settings.

## Analysis

The `!rfn` suffix in the detection name indicates a reputation-based file notification -- Defender's cloud ML model flagged files it had never seen before based on behavioral characteristics. This is the lowest-confidence detection type, distinct from signature matches (high confidence) and runtime behavioral detections (`!ml`, medium-high confidence).

The flagged scripts contained hardcoded SSH credentials in plaintext and executed remote commands via Paramiko:

```python
client = paramiko.SSHClient()
client.connect('192.168.1.186', username='flux', password='plaintext_password')
client.exec_command('sudo systemctl restart docker')
```

This pattern -- credential storage, remote execution, system modification, zero file reputation -- is behaviorally consistent with credential-stealing trojans. The ML classification was reasonable given the inputs.

## Determination

**False positive, high confidence.** The evidence:

The alert timestamps correlated directly with file development activity on the workstation. The flagged files resided in the user's project directory. The target IP (192.168.1.186) and username (flux) matched documented homelab infrastructure. Six Leonem-specific IOCs were checked, all negative. The detection type was reputation-based, not signature-based.

Time from detection to determination: approximately 25 minutes across four evidence sources -- the Defender console, file system metadata, infrastructure documentation, and open-source threat intelligence.

## Secondary Finding

The scripts contained hardcoded SSH passwords in plaintext across three files. This is a credential hygiene issue independent of the malware determination. If the workstation were compromised or the scripts committed to a public repository, these credentials would be immediately exposed.

The scripts triggered heuristic detection because they exhibit the same behavior as the malware they were flagged for: storing credentials in the clear and using them for remote access.

Remediation: migrate to SSH key-based authentication, remove all hardcoded credentials, audit version control history for prior exposure, and apply a targeted Defender exclusion scoped to the development directory.

## Key Points

Detection suffixes in Defender alert names encode the confidence level and methodology behind the classification. The suffix determines the appropriate triage posture -- `!rfn` warrants investigation, not immediate remediation.

Timestamp correlation between alert generation and file system activity is the fastest and most reliable evidence source for false positive determination on endpoint detections.

Establishing the IOC profile for a threat before making a determination is the difference between a documented conclusion and a guess.

False positive determinations should not terminate the investigation. The heuristic engine identified a genuine behavioral concern -- hardcoded credentials and remote execution -- that required remediation regardless of the malware classification.

---

*References:*
- [Microsoft Security Intelligence -- Trojan:Win32/Leonem](https://www.microsoft.com/en-us/wdsi/threats/malware-encyclopedia-description?Name=Trojan:Win32/Leonem)
- [Trend Micro -- Trojan.Win32.LEONEM](https://www.trendmicro.com/vinfo/us/threat-encyclopedia/malware/trojan.win32.leonem.vsnw17c23)
- [Gridinsoft -- Trojan:Win32/Leonem](https://gridinsoft.com/blogs/trojan-win32-leonem/)
