---
title: "Network Overview"
permalink: /docs/network/overview/
---

# Network Overview

## Subnet

- **Network:** 192.168.56.0/24
- **Type:** VirtualBox Host-Only Adapter
- **Purpose:** Isolated lab network for attack/defense practice

---

## Network Diagram

```
         192.168.56.0/24 (Host-Only Network)
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
   .1               .10               .50
 [HOST]           [DC01]            [WS01]
Your Laptop    Domain Ctrl      Win11 Target
                   │                 │
                   └────lab.local────┘
                          │
                         .102
                       [KALI]
                    Attack Machine
```

---

## IP Address Inventory

| IP | Hostname | OS | Role |
|----|----------|-----|------|
| 192.168.56.1 | Host | Windows | Your laptop (VirtualBox adapter) |
| 192.168.56.10 | DC01 | Windows Server 2022 | Domain Controller |
| 192.168.56.50 | WS01 | Windows 11 Pro | Target Workstation |
| 192.168.56.100 | - | - | VirtualBox DHCP server |
| 192.168.56.102 | Kali | Kali Linux 2025.4 | Attack Machine |
| 192.168.1.186 | secondbrain | Ubuntu 24.04 | SIEM Server (Splunk) |

---

## Domain Info

- **Domain Name:** lab.local
- **Domain Controller:** DC01 (192.168.56.10)
- **DNS Server:** DC01 (192.168.56.10)

---

## VirtualBox Network Types

| Type | Use Case | Internet | Reaches Home Network |
|------|----------|----------|---------------------|
| Host-Only | VM-to-VM communication | No | No |
| NAT | VM internet access | Yes | No |
| Bridged | VM on real network | Yes | Yes |

---

## Network Adapters by VM

### DC01

| Adapter | Type | IP | Purpose |
|---------|------|-----|---------|
| Ethernet | Host-Only | 192.168.56.10 | Lab network |
| Ethernet 2 | NAT | 10.0.3.15 | Internet access |
| Ethernet 3 | Bridged | 192.168.1.4 | Home network (SIEM) |

### WS01

| Adapter | Type | IP | Purpose |
|---------|------|-----|---------|
| Ethernet | Host-Only | 192.168.56.50 | Lab network |

### Kali

| Adapter | Type | IP | Purpose |
|---------|------|-----|---------|
| eth0 | Host-Only | 192.168.56.102 | Lab network |
| eth1 | NAT | 10.0.3.15 | Internet access |
