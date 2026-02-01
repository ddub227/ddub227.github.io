---
title: "Security Concepts"
permalink: /docs/learning/concepts/
---

# Security Concepts

Key concepts for understanding Active Directory attacks and defenses.

---

## Active Directory

### What is Active Directory?

Microsoft's directory service for Windows networks. Stores information about users, computers, groups, and policies. The Domain Controller (DC) is the server that runs AD.

### Key AD Components

- **Domain Controller (DC)** - Server that authenticates users and stores AD database
- **NTDS.dit** - The AD database file containing all password hashes
- **Kerberos** - Authentication protocol used by AD (port 88)
- **LDAP** - Protocol to query AD (port 389)
- **Group Policy (GPO)** - Centralized configuration management

### Why Attackers Target AD

- Compromise one DC = own all domain users
- Domain Admin = full control of entire network
- Password hashes stored centrally (NTDS.dit)

---

## Common Attacks

### Kerberoasting

**What:** Request service tickets for accounts with SPNs, crack them offline

**Why it works:** Service tickets encrypted with service account's password hash

**Detection:** Event ID 4769 with encryption type 0x17

### AS-REP Roasting

**What:** Request authentication for accounts without Kerberos pre-authentication

**Why it works:** Response encrypted with user's password hash

**Detection:** Event ID 4768 without pre-authentication

### Pass-the-Hash

**What:** Use captured NTLM hash to authenticate without knowing password

**Why it works:** Windows accepts hash directly for NTLM authentication

**Detection:** Sysmon Event ID 10 (lsass.exe access)

### LLMNR/NBNS Poisoning

**What:** Respond to broadcast name queries, capture credentials

**Why it works:** Windows broadcasts when DNS fails, trusts any response

**Tool:** Responder

**Detection:** Network traffic analysis

### DCSync

**What:** Impersonate a Domain Controller to request password hashes

**Why it works:** DCs sync with each other using replication protocol

**Requires:** Domain Admin or replication rights

**Detection:** Event ID 4662 with replication properties

---

## Defense Concepts

### Sysmon

Free Microsoft tool that logs detailed system activity:

- Event ID 1: Process creation (with command line!)
- Event ID 3: Network connections
- Event ID 10: Process access (detects credential dumping)
- Event ID 11: File creation

### SIEM (Security Information and Event Management)

Centralized log collection and analysis (e.g., Splunk, Security Onion)

### Defense in Depth

Multiple layers of security - if one fails, others protect

---

## Networking Concepts

### Common Ports

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | Linux remote access |
| 53 | DNS | Name resolution |
| 80 | HTTP | Web traffic |
| 88 | Kerberos | AD authentication |
| 135 | MSRPC | Windows RPC |
| 139 | NetBIOS | Legacy Windows |
| 389 | LDAP | Directory queries |
| 443 | HTTPS | Secure web |
| 445 | SMB | File sharing |
| 3389 | RDP | Remote Desktop |
| 5985 | WinRM | PowerShell remoting |

### CIDR Notation

- /24 = 255.255.255.0 = 256 addresses
- /16 = 255.255.0.0 = 65,536 addresses
- /8 = 255.0.0.0 = 16 million addresses

---

## Windows Event IDs

| Event ID | Source | Meaning |
|----------|--------|---------|
| 4624 | Security | Successful logon |
| 4625 | Security | Failed logon |
| 4662 | Security | Object access (DCSync detection) |
| 4768 | Security | Kerberos TGT request |
| 4769 | Security | Kerberos service ticket (Kerberoasting) |
| 7045 | System | New service installed |
