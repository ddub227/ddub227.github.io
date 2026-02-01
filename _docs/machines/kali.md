---
title: "Kali - Attack Platform"
permalink: /docs/machines/kali/
---

# Kali - Attack Machine

## Quick Info

| Property | Value |
|----------|-------|
| IP Address | 192.168.56.102 |
| Hostname | kali |
| OS | Kali Linux 2025.4 |
| Role | Attack machine |
| Credentials | kali / kali |

---

## Network Configuration

- **Adapter 1:** Host-only (192.168.56.0/24) - Lab network
- **Adapter 2:** NAT (10.0.3.0/24) - Internet access

---

## Key Tools Installed

### Network Scanning

- Nmap
- Masscan
- Netdiscover

### Active Directory

- BloodHound
- CrackMapExec
- Impacket (secretsdump, GetUserSPNs, psexec, etc.)
- Evil-WinRM
- Responder
- ldapsearch

### Credential Attacks

- Hashcat
- John the Ripper
- Hydra

### Web Testing

- Burp Suite
- Nikto
- Gobuster

### Exploitation

- Metasploit Framework
- SearchSploit

---

## Useful Commands

```bash
# Check IP
ip a

# Ping sweep
nmap -sn 192.168.56.0/24

# Full port scan
nmap -sV -p- --min-rate 1000 <target>

# Update Kali
sudo apt update && sudo apt upgrade -y

# LDAP enumeration
ldapsearch -x -H ldap://192.168.56.10 -b "dc=lab,dc=local"

# CrackMapExec user enum
crackmapexec smb 192.168.56.10 -u 'tuser' -p 'password' --users

# BloodHound collection
bloodhound-python -u 'tuser' -p 'password' -d lab.local -ns 192.168.56.10 -c all
```

---

## Lessons Learned

- Use `ip a` instead of `ifconfig` (deprecated)
- `-c 4` flag stops ping after 4 packets (Linux pings forever by default)
- Right arrow key accepts zsh autosuggestions
- Add DC to /etc/hosts for BloodHound DNS resolution
