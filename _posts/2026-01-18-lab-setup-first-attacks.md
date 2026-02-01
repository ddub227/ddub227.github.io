---
layout: post
title: "Building a Security Home Lab: Initial Setup and First Active Directory Attacks"
date: 2026-01-18
categories: [homelab, active-directory, pentesting]
tags: [bloodhound, crackmapexec, nmap, ldap, kali-linux, windows-server]
excerpt: "Complete walkthrough of setting up an Active Directory lab environment with Kali Linux attack platform, including reconnaissance, enumeration attacks, and network troubleshooting."
---

This post documents the setup of a cybersecurity home lab environment and the execution of initial Active Directory enumeration attacks. The lab consists of a Windows domain environment with a Domain Controller (DC01), a Windows 11 workstation (WS01), and a Kali Linux attack machine—all running in VirtualBox on an isolated host-only network.

<!--more-->

---

## Lab Architecture

### Network Topology

```
         192.168.56.0/24 (Host-Only Network)
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
   .10               .50              .102
 [DC01]            [WS01]           [KALI]
 Windows           Windows 11       Kali Linux
 Server 2022       Pro              2025.4
    │                 │                │
    └────lab.local────┘                │
           Domain                   Attack
         Controller                 Machine
```

### Virtual Machine Specifications

| Machine | OS | RAM | Role | IP Address |
|---------|-----|-----|------|------------|
| DC01 | Windows Server 2022 | 4 GB | Domain Controller, DNS | 192.168.56.10 |
| WS01 | Windows 11 Pro | 4 GB | Domain-joined Workstation | 192.168.56.50 |
| Kali | Kali Linux 2025.4 | 4 GB | Attack Platform | 192.168.56.102 |

### Network Configuration

All VMs utilize VirtualBox Host-Only Adapters for inter-VM communication. The Kali machine was later configured with a secondary NAT adapter to facilitate package updates while maintaining isolation on the primary interface.

```
Kali Network Interfaces:
├── eth0: 192.168.56.102/24 (Host-Only) - Lab network
└── eth1: 10.0.3.15/24 (NAT) - Internet access
```

---

## Phase 1: Attack Platform Configuration

### Kali Linux Installation

The Kali VM was imported from the official VirtualBox image available at kali.org. Post-import configuration included:

1. Modifying network adapter from NAT to Host-Only
2. Increasing RAM allocation from 2GB to 4GB
3. Adding secondary NAT adapter for package management

### Validating Network Connectivity

Initial connectivity validation was performed using ICMP and targeted port scanning:

```bash
# Discovery scan
nmap -sn 192.168.56.0/24

# Results: 5 hosts discovered
# 192.168.56.1   - VirtualBox Host Adapter
# 192.168.56.10  - DC01
# 192.168.56.50  - WS01
# 192.168.56.100 - VirtualBox DHCP Server
# 192.168.56.102 - Kali (self)
```

---

## Phase 2: Target Reconnaissance

### Full Port Scan of Domain Controller

A comprehensive port scan was executed against DC01 to identify the attack surface:

```bash
nmap -sV -p- --min-rate 1000 192.168.56.10
```

#### Results: 19 Open Ports

| Port | Service | Version |
|------|---------|---------|
| 53/tcp | DNS | Simple DNS Plus |
| 88/tcp | Kerberos | Microsoft Windows Kerberos |
| 135/tcp | MSRPC | Microsoft Windows RPC |
| 139/tcp | NetBIOS | netbios-ssn |
| 389/tcp | LDAP | Microsoft Windows AD LDAP |
| 445/tcp | SMB | microsoft-ds |
| 464/tcp | kpasswd5 | - |
| 593/tcp | RPC/HTTP | RPC over HTTP 1.0 |
| 636/tcp | LDAPS | tcpwrapped |
| 3268/tcp | Global Catalog | AD LDAP |
| 3269/tcp | GC-SSL | tcpwrapped |
| 5985/tcp | WinRM | Microsoft HTTPAPI 2.0 |
| 9389/tcp | AD Web Services | .NET Message Framing |
| 49664-49691/tcp | Dynamic RPC | Microsoft Windows RPC |

#### Key Attack Vectors Identified

- **Port 88 (Kerberos)**: Kerberoasting, AS-REP Roasting
- **Port 389 (LDAP)**: User enumeration, BloodHound collection
- **Port 445 (SMB)**: Relay attacks, PsExec, credential testing
- **Port 5985 (WinRM)**: Remote PowerShell execution via Evil-WinRM

### Workstation Scan Results

WS01 presented a more hardened profile due to Windows Firewall:

```bash
nmap -Pn -p 135,445,3389 192.168.56.50

# Results:
# 135/tcp  open     msrpc
# 445/tcp  filtered microsoft-ds
# 3389/tcp filtered ms-wbt-server
```

Notable finding: ICMP (ping) was blocked, requiring the `-Pn` flag to skip host discovery.

---

## Phase 3: Active Directory Enumeration

### LDAP User Enumeration

The first enumeration technique employed was LDAP querying. Anonymous bind was blocked (good security practice), requiring authenticated queries:

```bash
ldapsearch -x -H ldap://192.168.56.10 \
  -D "tuser@lab.local" -W \
  -b "dc=lab,dc=local" \
  "(objectClass=user)" sAMAccountName | grep sAMAccountName
```

#### Results

```
sAMAccountName: Administrator
sAMAccountName: Guest
sAMAccountName: DC01$
sAMAccountName: krbtgt
sAMAccountName: tuser
sAMAccountName: WS01$
```

#### Account Analysis

| Account | Type | Notes |
|---------|------|-------|
| Administrator | Domain Admin | High-value target |
| Guest | Built-in | Typically disabled |
| DC01$ | Computer | Domain Controller machine account |
| krbtgt | Service | Kerberos TGT account (Golden Ticket target) |
| tuser | User | Test user account |
| WS01$ | Computer | Workstation machine account |

The `$` suffix denotes computer accounts in Active Directory.

### CrackMapExec Enumeration

CrackMapExec provided more detailed enumeration with credential validation:

```bash
crackmapexec smb 192.168.56.10 -u tuser -p 'S@nford22' --users
```

#### Output Analysis

```
SMB  192.168.56.10  445  DC01  [*] Windows Server 2022 Build 20348 x64
SMB  192.168.56.10  445  DC01  [+] lab.local\tuser:S@nford22
SMB  192.168.56.10  445  DC01  [+] Enumerated domain user(s)
SMB  192.168.56.10  445  DC01  lab.local\tuser          badpwdcount: 0
SMB  192.168.56.10  445  DC01  lab.local\krbtgt         badpwdcount: 0
SMB  192.168.56.10  445  DC01  lab.local\Guest          badpwdcount: 0
SMB  192.168.56.10  445  DC01  lab.local\Administrator  badpwdcount: 0
```

Key information extracted:
- **OS Version**: Windows Server 2022 Build 20348 x64
- **SMB Signing**: True (complicates relay attacks)
- **SMBv1**: Disabled (good security posture)
- **badpwdcount**: 0 for all accounts (no lockout risk)

---

## Phase 4: BloodHound Data Collection

### Installation and Dependencies

```bash
sudo apt update && sudo apt install bloodhound neo4j -y
```

### Starting the Neo4j Database

```bash
sudo neo4j console
# Wait for "Started." message
```

### Data Collection with BloodHound-Python

Initial collection attempts failed due to DNS resolution issues:

```
dns.resolver.LifetimeTimeout: The resolution lifetime expired
ERROR: Could not find a domain controller
```

#### Resolution

1. Added static host entry:
```bash
echo "192.168.56.10 DC01.lab.local DC01 lab.local" | sudo tee -a /etc/hosts
```

2. Executed collection with explicit parameters:
```bash
bloodhound-python -u tuser -p 'S@nford22' \
  -d lab.local \
  -ns 192.168.56.10 \
  -dc DC01.lab.local \
  -c all \
  --disable-autogc
```

#### Successful Collection Results

```
INFO: Found AD domain: lab.local
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Found 5 users
INFO: Found 52 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Done in 00M 02S
```

---

## Phase 5: Network Troubleshooting Deep Dive

A significant portion of the lab session involved troubleshooting network connectivity issues. This section documents the methodology employed.

### Symptom

BloodHound-python reported DNS timeouts despite successful CrackMapExec execution earlier in the session.

### Diagnostic Process

#### Layer 3 Testing (Network)

```bash
ping -c 2 192.168.56.10
# Result: Destination Host Unreachable (100% packet loss)
```

#### Layer 4 Testing (Transport)

```bash
nc -zv 192.168.56.10 445
# Result: DC01.lab.local [192.168.56.10] 445 (microsoft-ds) open
```

#### ARP Cache Verification

```bash
arp -a
# DC01.lab.local (192.168.56.10) at 08:00:27:bd:c6:37 [ether] on eth0
```

#### Routing Table Analysis

```bash
ip route
# default via 10.0.3.2 dev eth1 proto dhcp metric 101
# 10.0.3.0/24 dev eth1 proto kernel scope link metric 101
# 192.168.56.0/24 dev eth0 proto kernel scope link metric 100
```

#### Bidirectional Connectivity Test

From DC01 to Kali:
```cmd
ping 192.168.56.102
# Reply from 192.168.56.102: bytes=32 time=1ms TTL=64
# 4 packets transmitted, 4 received, 0% loss
```

### Root Cause Analysis

| Test | Result | Conclusion |
|------|--------|------------|
| Kali → DC01 ICMP | Failed | ICMP blocked |
| Kali → DC01 TCP/445 | Success | TCP works |
| DC01 → Kali ICMP | Success | Network functional |
| ARP Resolution | Success | Layer 2 functional |
| Routing | Correct | Layer 3 configured properly |

**Conclusion**: Windows Firewall on DC01 blocks inbound ICMP but permits TCP connections. The DNS SRV query timeouts were caused by the combination of ICMP-based health checks and aggressive timeout values, not actual DNS misconfiguration.

### Key Lesson

Never rely solely on ICMP (ping) for connectivity validation. Always test specific TCP/UDP ports relevant to your use case.

---

## Security Tools Summary

| Tool | Purpose | Command Example |
|------|---------|-----------------|
| Nmap | Port scanning, service detection | `nmap -sV -p- <target>` |
| ldapsearch | LDAP queries | `ldapsearch -x -H ldap://<dc>` |
| CrackMapExec | AD enumeration, credential testing | `crackmapexec smb <target> --users` |
| BloodHound-python | AD relationship collection | `bloodhound-python -c all` |
| netcat (nc) | TCP/UDP connectivity testing | `nc -zv <target> <port>` |

---

## Technical Concepts Reference

### Active Directory Account Types

- **User Accounts**: Human users (e.g., `tuser`)
- **Computer Accounts**: Domain-joined machines, identified by `$` suffix (e.g., `WS01$`)
- **Service Accounts**: Accounts for services (e.g., `krbtgt` for Kerberos)

### sAMAccountName Attribute

The `sAMAccountName` (Security Account Manager Account Name) is a legacy attribute from Windows NT. The lowercase 's' follows LDAP's lowerCamelCase naming convention where the first letter of an attribute is always lowercase.

### badpwdcount Attribute

Tracks consecutive failed authentication attempts. Resets upon successful authentication or after a policy-defined timeout. Critical for:
- Avoiding account lockouts during password spraying
- Identifying accounts under active attack

### DNS SRV Records in Active Directory

Active Directory relies on DNS SRV records for service discovery:
- `_ldap._tcp.dc._msdcs.<domain>` - Domain Controller location
- `_kerberos._tcp.<domain>` - Kerberos KDC location
- `_gc._tcp.<domain>` - Global Catalog location

These cannot be resolved via `/etc/hosts` entries, requiring actual DNS server queries.

---

## Conclusion

This lab session established a functional Active Directory attack environment and demonstrated fundamental enumeration techniques. Key accomplishments include:

1. Complete network reconnaissance of the domain environment
2. Successful user enumeration via multiple vectors (LDAP, SMB)
3. BloodHound data collection for attack path analysis
4. Practical troubleshooting experience with network connectivity issues

The next phase will involve importing BloodHound data for visualization and executing credential-based attacks such as Kerberoasting.

---

## References

- [BloodHound Documentation](https://bloodhound.readthedocs.io/)
- [CrackMapExec Wiki](https://wiki.porchetta.industries/)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [Microsoft AD DS Documentation](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/)
