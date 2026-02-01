---
layout: post
title: "Deploying Splunk SIEM: From Discovery to Log Forwarding"
date: 2026-01-25
categories: [homelab, siem, infrastructure]
tags: [splunk, docker, windows-server, linux, logging]
excerpt: "Discovered an old laptop, turned it into a dedicated SIEM server, deployed Splunk Enterprise in Docker, and configured Windows Event Log forwarding from the domain controller."
---

Today's session took an unexpected turn - I discovered a forgotten Linux server on my home network and turned it into a dedicated SIEM platform. This post documents deploying Splunk Enterprise in Docker and configuring Windows Event Log forwarding from DC01.

<!--more-->

---

## Summary

| Field | Value |
|-------|-------|
| **Focus** | Infrastructure |
| **Systems Used** | secondbrain (Linux server), DC01, Splunk, Docker |
| **Goal** | Deploy Splunk SIEM on dedicated hardware and configure log forwarding |
| **Outcome** | Achieved |

---

## Infrastructure Discovery

### The "secondbrain" Server

Found an existing Linux server on my home network that was set up months ago and forgotten:

| Component | Value |
|-----------|-------|
| Hostname | secondbrain |
| CPU | Intel Core i7-7500U @ 2.70GHz (4 cores) |
| RAM | 8 GB |
| Disk | 1 TB HDD (Seagate ST1000LM035) |
| OS | Ubuntu 24.04.3 LTS |
| Hardware | Acer Aspire A515-51 |

This solves a resource constraint - offloading SIEM workload from my main laptop (16GB RAM) to dedicated hardware (8GB RAM).

---

## Docker Installation

Set up Docker on the secondbrain server:

```bash
sudo apt install docker.io -y
sudo usermod -aG docker flux
docker --version  # Docker version 28.2.2
```

---

## Splunk Deployment

### Initial Deployment Issues

First attempt at running Splunk container kept crashing:
- Container exited immediately with code 1
- `docker ps` showed no running containers

**Root Cause:** Splunk updated licensing requirements - now requires TWO acceptance flags instead of one.

### Working Deployment Command

```bash
docker run -d --name splunk -p 8000:8000 -p 9997:9997 \
  -e SPLUNK_GENERAL_TERMS='--accept-sgt-current-at-splunk-com' \
  -e SPLUNK_START_ARGS='--accept-license' \
  -e SPLUNK_PASSWORD='Changeme123!' \
  splunk/splunk:latest
```

**Key lesson:** Always check container logs with `docker logs <name>` when containers crash.

---

## Server Configuration

### Lid Switch for Headless Operation

Configured the laptop to stay running with lid closed:

```bash
# Edit /etc/systemd/logind.conf
HandleLidSwitch=ignore

# Apply changes
sudo systemctl restart systemd-logind
```

---

## DC01 Network Configuration

The domain controller needed additional network adapters to reach the SIEM server:

| Adapter | Type | IP | Purpose |
|---------|------|-----|---------|
| Ethernet | Host-Only | 192.168.56.10 | Lab network |
| Ethernet 2 | NAT | 10.0.3.15 | Internet access |
| Ethernet 3 | Bridged | 192.168.1.4 | Home network (reaches secondbrain) |

---

## Universal Forwarder Setup

### Installation on DC01

Downloaded and installed Splunk Universal Forwarder on DC01, then created the inputs configuration:

**Location:** `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

```ini
[WinEventLog://Security]
disabled = 0
index = main

[WinEventLog://System]
disabled = 0
index = main

[WinEventLog://Application]
disabled = 0
index = main
```

---

## Troubleshooting

### Problem: DC01 Couldn't Reach secondbrain on Port 9997

**Symptoms:**
- `Test-NetConnection 192.168.1.186 -Port 9997` failed
- Logs not appearing in Splunk

**Diagnosis:**
1. DC01 only had Host-Only (192.168.56.x) and NAT (10.0.x.x) adapters
2. Neither could route to home network (192.168.1.x)

**Solution:** Added Bridged Adapter to DC01, connected to laptop's WiFi.

### Problem: Port 9997 Still Unreachable

**Symptoms:**
- DC01 had 192.168.1.4 address (correct network)
- Still couldn't connect to 192.168.1.186:9997

**Root Cause:** Docker containers are isolated - forgot to expose port 9997 with `-p 9997:9997`

**Solution:**
```bash
docker stop splunk && docker rm splunk
docker run -d --name splunk -p 8000:8000 -p 9997:9997 ...
```

**Key lesson:** Docker port mapping must be explicit - internal container ports are invisible without `-p`.

---

## Concepts Learned

### Docker Images vs Containers

- **Image:** A frozen snapshot/template (like a recipe)
- **Container:** A running instance of an image (like the actual dish)
- Images are downloaded from Docker Hub, containers are created from images

### Docker Port Mapping

- `-p 8000:8000` means: host port 8000 → container port 8000
- Without explicit mapping, container ports are inaccessible from outside
- Multiple ports: `-p 8000:8000 -p 9997:9997`

### VirtualBox Network Types

| Type | Use Case | Internet | Reaches Home Network |
|------|----------|----------|---------------------|
| Host-Only | VM-to-VM communication | No | No |
| NAT | VM internet access | Yes | No |
| Bridged | VM on real network | Yes | Yes |

### Splunk Architecture

```
┌─────────────────┐                    ┌─────────────────┐
│  Windows VM     │                    │   SIEM Server   │
│                 │    Port 9997       │                 │
│  Universal   ─────────────────────►  │    Splunk       │
│  Forwarder      │     (logs)         │   Enterprise    │
│  (lightweight)  │                    │   (full)        │
└─────────────────┘                    └─────────────────┘
     ~50 MB                                 ~1 GB
     Collects & sends                  Stores & searches
```

---

## Commands Reference

### Check System Specs (Linux)

```bash
hostnamectl          # OS and hostname info
free -h              # RAM usage (human readable)
df -h                # Disk usage (human readable)
lscpu | grep -E "Model name|^CPU\(s\):"  # CPU info
lsblk -d -o NAME,ROTA,SIZE,MODEL  # Disk type (ROTA=1 means HDD)
```

### Docker Container Management

```bash
docker run -d --name splunk ...  # Start container in background
docker ps                        # List running containers
docker ps -a                     # List ALL containers (including stopped)
docker logs splunk               # View container output/errors
docker stop splunk               # Stop container
docker rm splunk                 # Remove container
```

### Windows Port Testing

```powershell
Test-NetConnection 192.168.1.186 -Port 9997
```

---

## Configuration Changes Summary

| System | Change | Purpose |
|--------|--------|---------|
| secondbrain | Installed Docker | Container runtime for Splunk |
| secondbrain | `HandleLidSwitch=ignore` | Keep running with lid closed |
| secondbrain | Splunk container with ports 8000, 9997 | SIEM deployment |
| DC01 | Added NAT adapter (Adapter 2) | Internet access for downloads |
| DC01 | Added Bridged adapter (Adapter 3) | Reach home network (secondbrain) |
| DC01 | Installed Universal Forwarder | Send logs to Splunk |
| DC01 | Created inputs.conf | Collect Security, System, Application logs |

---

## Skills Practiced

| Skill | Activity | SOC Relevance |
|-------|----------|---------------|
| SIEM Deployment | Installed Splunk Enterprise | Core SOC tool |
| Log Forwarding | Configured Universal Forwarder | How logs get into SIEM |
| Docker | Deployed containerized application | Modern infrastructure skill |
| Linux Administration | SSH, package management, service config | Server management |
| Network Troubleshooting | Diagnosed connectivity issues | Daily SOC skill |
| Windows Administration | PowerShell, services, network config | Endpoint knowledge |

---

## Next Steps

- [ ] Verify logs are flowing from DC01 to Splunk
- [ ] Run first Splunk search query
- [ ] Install Universal Forwarder on WS01
- [ ] Create basic detection rules
- [ ] **PRIORITY: Harden network before any internet exposure**

---

## Key Takeaway

> A dedicated Linux server (even an old laptop) dramatically improves homelab capabilities. By offloading Splunk to secondbrain, the main laptop's resources are freed for running VMs, and the SIEM can run 24/7 independently. Physical hardware separation also mirrors real enterprise architecture where SIEM servers are dedicated systems.
