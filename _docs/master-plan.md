---
title: "Master Plan"
permalink: /docs/master-plan/
---

# Cybersecurity Home Lab - Master Plan

## Current Progress

| Phase | Status | Last Updated |
|-------|--------|--------------|
| Phase 1: DC01 | Complete | 2026-01-05 |
| Phase 2.1: WS01 (Win11) | Complete | 2026-01-18 |
| Phase 2.2: WS02 (Win11) | Not Started | - |
| Phase 2.3: Ubuntu01 | Not Started | - |
| Phase 3: Splunk SIEM | Complete | 2026-01-25 |
| Phase 4: Kali Linux | Complete | 2026-01-18 |
| Phase 5: Vulnerable VMs | Not Started | - |
| Phase 6: pfSense | Not Started | - |

---

## Network Architecture Overview

```
                              [INTERNET]
                                  |
                           [pfSense Firewall]
                             192.168.1.1
                                  |
          +-----------------------+-----------------------+
          |                       |                       |
     [MANAGEMENT]            [CORPORATE]             [ATTACK]
     192.168.56.0/24        192.168.10.0/24        192.168.99.0/24
          |                       |                       |
     +----+----+           +------+------+               |
     |         |           |      |      |               |
  [DC01]   [Splunk]    [Win11-1] [Win11-2] [Ubuntu]     [Kali]
    .10      .20           .50      .51      .52         .10
```

---

## Phase 1: Foundation (Complete)

### DC01 - Domain Controller

- [x] Windows Server 2022
- [x] Active Directory Domain Services
- [x] DNS Server
- [x] IP: 192.168.56.10
- [x] Domain: lab.local
- [x] Test user: tuser@lab.local

---

## Phase 2: Corporate Network

### 2.1 Windows 11 Workstation (Primary Target) - Complete

| Setting | Value |
|---------|-------|
| Name | WS01 |
| OS | Windows 11 Pro |
| RAM | 4 GB |
| Disk | 64 GB |
| IP | 192.168.56.50 |
| DNS | 192.168.56.10 (DC01) |
| Domain | lab.local |

**VirtualBox Requirements for Windows 11:**
- VirtualBox 7.0+ required
- Enable EFI: Settings > System > Enable EFI
- Enable TPM: Settings > System > TPM (v2.0)
- 4+ GB RAM, 2+ CPUs

### 2.2 Windows 11 Workstation (Secondary - Optional)

| Setting | Value |
|---------|-------|
| Name | WS02 |
| OS | Windows 11 Pro |
| RAM | 4 GB |
| Disk | 64 GB |
| IP | 192.168.56.51 |

### 2.3 Ubuntu Server

| Setting | Value |
|---------|-------|
| Name | UBUNTU01 |
| OS | Ubuntu Server 22.04 LTS |
| RAM | 2 GB |
| Disk | 40 GB |
| IP | 192.168.56.52 |
| Purpose | Web server, SSH target |

---

## Phase 3: Security Monitoring (SIEM) - Complete

### Splunk Enterprise

| Setting | Value |
|---------|-------|
| Name | secondbrain |
| OS | Ubuntu 24.04 LTS |
| RAM | 8 GB |
| Disk | 1 TB |
| IP | 192.168.1.186 |
| Web UI | http://192.168.1.186:8000 |
| Deployment | Docker container |

**Log Sources Configured:**
- DC01: Security Events, System Events, Application Events

---

## Phase 4: Attack Infrastructure - Complete

### Kali Linux

| Setting | Value |
|---------|-------|
| Name | KALI |
| OS | Kali Linux 2025.4 |
| RAM | 4 GB |
| Disk | 80 GB |
| IP | 192.168.56.102 |
| Network | Host-Only + NAT |

**Essential Tools Pre-installed:**
- Nmap, Metasploit, Burp Suite, Wireshark
- Responder, BloodHound, Impacket, CrackMapExec

---

## Phase 5: Vulnerable Targets (Planned)

| VM | Purpose | Download |
|----|---------|----------|
| Metasploitable 2 | Linux exploitation | Rapid7 |
| DVWA | Web app vulnerabilities | Docker/VM |
| Vulnhub VMs | CTF-style challenges | vulnhub.com |

---

## Phase 6: Network Segmentation (Planned)

### pfSense Firewall

| Setting | Value |
|---------|-------|
| Name | FIREWALL |
| OS | pfSense CE |
| RAM | 2 GB |
| Disk | 20 GB |
| WAN | NAT/Bridged |
| LAN | 192.168.56.1 |

---

## Resource Summary

| VM | RAM | Disk | Status |
|----|-----|------|--------|
| DC01 | 4 GB | 60 GB | Deployed |
| WS01 | 4 GB | 64 GB | Deployed |
| Kali | 4 GB | 80 GB | Deployed |
| Splunk (secondbrain) | 8 GB | 1 TB | Deployed |
| Ubuntu01 | 2 GB | 40 GB | Planned |
| WS02 | 4 GB | 64 GB | Planned |
| pfSense | 2 GB | 20 GB | Planned |

---

## Attack Scenarios to Practice

| Scenario | Tools | Detection |
|----------|-------|-----------|
| Network Reconnaissance | Nmap | Firewall logs |
| Password Spraying | CrackMapExec | Windows Security 4625 |
| Kerberoasting | Impacket | Windows Security 4769 |
| Pass-the-Hash | Mimikatz | Sysmon Event 10 |
| LLMNR Poisoning | Responder | Network traffic |
| BloodHound Enumeration | SharpHound | Sysmon Event 1 |
| DCSync Attack | Mimikatz | Windows Security 4662 |

---

## Detection Rules to Build

| Attack | Splunk Query Focus |
|--------|-------------------|
| Brute Force | EventCode=4625 count > 5 in 1min |
| Kerberoasting | EventCode=4769 TicketEncryptionType=0x17 |
| DCSync | EventCode=4662 Properties contains Replication |
| Pass-the-Hash | Sysmon EventID=10 TargetImage=lsass.exe |
| New Service | EventCode=7045 |
| Scheduled Task | EventCode=4698 |

---

## Useful Downloads

| Software | URL |
|----------|-----|
| Windows 11 ISO | microsoft.com/evalcenter |
| Windows Server 2022 | microsoft.com/evalcenter |
| Kali Linux | kali.org/get-kali |
| Ubuntu Server | ubuntu.com/download/server |
| Splunk Enterprise | splunk.com/download |
| pfSense | pfsense.org/download |
| Sysmon | docs.microsoft.com/sysinternals |
