---
layout: post
title: "API Fundamentals: Splunk REST API with Secure Credential Management"
date: 2026-02-03
categories: [homelab, api, blue-team]
tags: [splunk, rest-api, bitwarden, ssl, python, automation, soc]
excerpt: "Configured Splunk REST API with HTTPS and integrated Bitwarden CLI for secure credential management - essential skills for SOC automation and SIEM integration."
---

Today's session focused on **API fundamentals** and **secure credential management** - critical skills for SOC automation. Successfully configured Splunk's REST API with HTTPS and integrated Bitwarden CLI for secure password retrieval, eliminating hardcoded credentials from scripts.

<!--more-->

---

## Session Overview

| Field | Value |
|-------|-------|
| **Focus** | REST API fundamentals, HTTPS/SSL configuration, secure credential management, Python automation |
| **Systems Used** | Splunk Enterprise, Home Server (Ubuntu/Docker), Bitwarden CLI, OpenSSL |
| **Goal** | Configure Splunk REST API with HTTPS and integrate secure credential retrieval via Bitwarden CLI |

---

## Key Accomplishments

- **HTTPS Configuration:** Generated self-signed SSL certificates with OpenSSL and configured Splunk to use them
- **REST API Access:** Exposed Splunk's REST API port (8089) in Docker and verified connectivity
- **Secure Credentials:** Integrated Bitwarden CLI for runtime password retrieval - no hardcoded secrets
- **Python Automation:** Created multiple scripts for API testing, search queries, and security alerts
- **Troubleshooting:** Diagnosed SSL errors, port confusion (8000 vs 8089), and Docker container issues

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Bitwarden CLI | Secure credential retrieval at runtime |
| OpenSSL | SSL certificate generation |
| Splunk REST API | Programmatic SIEM access |
| Docker | Container management |
| Python | API scripting |
| SSH | Remote server management |

---

## REST API Fundamentals

REST (Representational State Transfer) is an architectural style for web services. APIs expose endpoints that accept HTTP requests and return data (usually JSON).

### Key Concepts

| Concept | Description |
|---------|-------------|
| **HTTP Methods** | GET (retrieve), POST (create), PUT (update), DELETE (remove) |
| **Status Codes** | 200 (OK), 201 (Created), 401 (Unauthorized), 404 (Not Found) |
| **Authentication** | Basic auth sends username:password base64-encoded in header |

### Splunk Port Architecture

| Port | Purpose |
|------|---------|
| 8000 | Web UI (Splunk Web) |
| 8089 | REST API (management interface) |
| 9997 | Forwarding (receives logs from Universal Forwarders) |

---

## SSL/TLS Certificate Configuration

Generated self-signed certificates for internal HTTPS:

```bash
# Generate self-signed certificate (valid 365 days)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout splunk.key -out splunk.crt \
  -subj "/CN=splunk/O=homelab"
```

Configured Splunk via `web.conf`:

```ini
[settings]
enableSplunkWebSSL = true
privKeyPath = /opt/splunk/etc/auth/mycerts/splunk.key
serverCert = /opt/splunk/etc/auth/mycerts/splunk.crt
```

---

## Secure Credential Management with Bitwarden CLI

### Why Not Hardcode Credentials?

| Method | Security Level | Risk |
|--------|---------------|------|
| Hardcoded in scripts | Terrible | Anyone with file access gets credentials |
| Environment variables | Better | Still readable by anyone with file access |
| Secret managers (Bitwarden) | Best | Encrypted vault, requires authentication |

### Bitwarden CLI Workflow

```bash
# Install via winget
winget install Bitwarden.CLI

# Login and unlock vault
bw login
bw unlock

# Set session key (required for commands)
$env:BW_SESSION = "<session_key>"

# Retrieve password at runtime
bw get password splunk.com
```

### Python Integration

```python
def get_password_from_bitwarden(item_name):
    result = subprocess.run(
        ["bw", "get", "password", item_name],
        capture_output=True,
        text=True,
        timeout=30
    )
    if result.returncode == 0:
        return result.stdout.strip()
    return None
```

---

## API Connection Test Script

Created a secure connection test script that retrieves credentials from Bitwarden:

```python
import requests
import subprocess

# Get password securely
password = get_password_from_bitwarden("splunk.com")

# Test API connection
url = "https://192.168.1.186:8089/services/server/info"
response = requests.get(
    url,
    auth=("admin", password),
    verify=False,  # Self-signed cert
    params={"output_mode": "json"}
)

if response.status_code == 200:
    data = response.json()
    print(f"Server: {data['entry'][0]['content']['serverName']}")
    print(f"Version: {data['entry'][0]['content']['version']}")
```

---

## Troubleshooting Lessons

### SSL WRONG_VERSION_NUMBER Error

**Cause:** HTTPS request hitting HTTP-only endpoint
**Fix:** Ensure port/protocol match (HTTPS → 8089 with SSL configured)

### 404 Not Found on API Endpoint

**Cause:** Wrong port - using 8000 (Web UI) instead of 8089 (API)
**Fix:** Use port 8089 for REST API calls

### Docker Container Restart Loop (Exit 137)

**Cause:** Mounting single file (web.conf) caused "Device busy" error
**Fix:** Mount directories only; copy files after container start

---

## Scripts Created

| Script | Purpose |
|--------|---------|
| `01_splunk_connection_test.py` | Secure API connection using Bitwarden |
| `00_network_diagnostic.py` | Systematic network troubleshooting |
| `splunk_api_test.py` | Basic API connectivity test |
| `splunk_api_search.py` | Run SPL queries via API |
| `splunk_api_alerts.py` | Security alert detection functions |

---

## SOC Relevance

These skills directly apply to Tier 1 SOC analyst work:

- **API Integration:** Automate alert enrichment, threat intel lookups, report generation
- **Secure Credentials:** Enterprise environments require proper secrets management
- **Troubleshooting:** Systematic diagnosis under pressure is essential
- **Scripting:** Automating repetitive tasks increases efficiency

---

## Next Steps

- [ ] Run SPL searches programmatically via API
- [ ] Build automated security alert scripts (failed logins, Kerberoasting)
- [ ] Integrate threat intelligence APIs (VirusTotal, AbuseIPDB)
- [ ] Create automated daily security reports
- [ ] Build Splunk dashboard via API

---

## Key Takeaway

> **Insight:** Secure credential management isn't optional - it's foundational. Real-world automation requires APIs, and APIs require credentials. Bitwarden CLI provides enterprise-grade security for a home lab, teaching habits that transfer directly to production environments.
