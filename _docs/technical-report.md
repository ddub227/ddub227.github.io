---
title: "Technical Report"
permalink: /docs/technical-report/
---

# HomeLab - Technical Report

> Cybersecurity Active Directory lab environment for red team/blue team training

**Last Updated:** 2026-01-25
**Status:** Phase 4 Complete (4/7 Systems Operational)

---

## Network Architecture

| Component | Specification |
|-----------|---------------|
| **Primary Subnet** | 192.168.56.0/24 (VirtualBox Host-Only) |
| **Domain** | lab.local |
| **Hypervisor** | Oracle VirtualBox |
| **Host Gateway** | 192.168.56.1 (Windows) |
| **SIEM Server** | 192.168.1.186 (Home network) |

```
┌─────────────────────────────────────────────────────────────────┐
│                    192.168.56.0/24 Network                      │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │    DC01      │  │    WS01      │  │    Kali      │          │
│  │  .56.10      │  │   .56.50     │  │   .56.102    │          │
│  │  Win Server  │  │   Win 11     │  │   Linux      │          │
│  │  2022        │  │   Pro        │  │   2025.4     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         │                 │                 │                   │
│         └─────────────────┼─────────────────┘                   │
│                           │                                     │
│                    ┌──────┴──────┐                              │
│                    │   Host      │                              │
│                    │  .56.1      │                              │
│                    └─────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Deployed Infrastructure

### DC01 - Domain Controller

| Attribute | Value |
|-----------|-------|
| **OS** | Windows Server 2022 |
| **IP Address** | 192.168.56.10 |
| **Hostname** | DC01.lab.local |
| **RAM** | 4 GB |
| **Disk** | 60 GB |

**Roles & Features:**
- Active Directory Domain Services (AD DS)
- DNS Server (integrated)
- Kerberos Key Distribution Center (KDC)
- Global Catalog Server
- Group Policy Management

**Exposed Services (19 Ports):**

| Port | Service |
|------|---------|
| 53 | DNS |
| 88 | Kerberos |
| 135 | MS-RPC |
| 139 | NetBIOS Session |
| 389 | LDAP |
| 445 | SMB |
| 464 | Kerberos Password |
| 636 | LDAPS |
| 3268 | Global Catalog |
| 5985 | WinRM |
| 9389 | AD Web Services |

**Domain Accounts:**
- `Administrator` - Domain Admin
- `tuser` - Standard Domain User

---

### WS01 - Windows Workstation

| Attribute | Value |
|-----------|-------|
| **OS** | Windows 11 Pro |
| **IP Address** | 192.168.56.50 |
| **Hostname** | WS01.lab.local |
| **RAM** | 4 GB |
| **Disk** | 64 GB |

**Configuration:**
- Domain-joined to `lab.local`
- DNS: 192.168.56.10 (DC01)
- VirtualBox EFI boot enabled
- TPM v2.0 emulation enabled

**Security Tooling:**
- **Sysmon** with SwiftOnSecurity configuration
  - Process creation logging (Event ID 1)
  - Network connection logging (Event ID 3)
  - File creation logging (Event ID 11)
  - Registry modification logging (Event ID 13)

---

### Kali - Attack Platform

| Attribute | Value |
|-----------|-------|
| **OS** | Kali Linux 2025.4 |
| **IP Address** | 192.168.56.102 |
| **RAM** | 4 GB |
| **Disk** | 80 GB |

**Attack Tooling:**

| Category | Tools |
|----------|-------|
| **Reconnaissance** | Nmap, Masscan, Netdiscover |
| **AD Enumeration** | BloodHound, ldapsearch, enum4linux |
| **Credential Attacks** | CrackMapExec, Hashcat, John the Ripper, Hydra |
| **Exploitation** | Metasploit Framework, Impacket Suite |
| **Remote Access** | Evil-WinRM, psexec.py, wmiexec.py |
| **Network Attacks** | Responder |

---

### secondbrain - SIEM Server

| Attribute | Value |
|-----------|-------|
| **OS** | Ubuntu 24.04.3 LTS |
| **IP Address** | 192.168.1.186 |
| **RAM** | 8 GB |
| **Disk** | 1 TB HDD |
| **Hardware** | Acer Aspire A515-51 |

**Deployment:**
- Splunk Enterprise in Docker container
- Ports 8000 (web UI) and 9997 (forwarder) exposed
- Receives logs from DC01 via Universal Forwarder

---

## Resource Summary

| VM | RAM | Disk | Status |
|----|-----|------|--------|
| DC01 | 4 GB | 60 GB | Deployed |
| WS01 | 4 GB | 64 GB | Deployed |
| Kali | 4 GB | 80 GB | Deployed |
| secondbrain | 8 GB | 1 TB | Deployed |
| **Total Deployed** | **20 GB** | **1.2 TB** | - |

---

## Attack Scenarios

| Attack | Tools | Detection |
|--------|-------|-----------|
| Network Reconnaissance | Nmap | Firewall logs |
| Password Spraying | CrackMapExec | Event ID 4625 |
| Kerberoasting | GetUserSPNs.py | Event ID 4769 (0x17) |
| AS-REP Roasting | GetNPUsers.py | Event ID 4768 |
| Pass-the-Hash | Mimikatz | Sysmon Event 10 |
| LLMNR Poisoning | Responder | Network analysis |
| BloodHound Collection | SharpHound | Sysmon Event 1 |
| DCSync | secretsdump.py | Event ID 4662 |
| Lateral Movement | psexec.py | Event ID 4624 Type 3 |

---

## Detection Rules (Splunk)

```spl
# Brute Force Detection
index=windows EventCode=4625
| stats count by src_ip
| where count > 5

# Kerberoasting Detection
index=windows EventCode=4769 Ticket_Encryption_Type=0x17
| table _time, Account_Name, Service_Name, Client_Address

# DCSync Attack Detection
index=windows EventCode=4662
  Properties="*1131f6aa-9c07-11d1-f79f-00c04fc2dcd2*"
| table _time, Account_Name, Object_Name

# Pass-the-Hash Detection
index=sysmon EventCode=10 TargetImage="*lsass.exe"
| table _time, SourceImage, SourceUser, GrantedAccess
```

---

## Project Roadmap

| Phase | Description | Status |
|-------|-------------|--------|
| 1 | Domain Controller (DC01) | Complete |
| 2 | Primary Workstation (WS01) | Complete |
| 3 | Sysmon Deployment | Complete |
| 4 | Attack Platform (Kali) | Complete |
| 5 | SIEM (Splunk) | Complete |
| 6 | Additional Targets (Ubuntu, WS02) | Planned |
| 7 | Network Segmentation (pfSense) | Planned |

---

## Technologies

- **Virtualization:** Oracle VirtualBox
- **Operating Systems:** Windows Server 2022, Windows 11 Pro, Kali Linux 2025.4, Ubuntu 24.04 LTS
- **Directory Services:** Active Directory Domain Services
- **Monitoring:** Sysmon, Splunk Enterprise
- **Containerization:** Docker
- **Documentation:** Obsidian, Jekyll
