This is my personal infrastructure lab for hands-on learning and skill development. I document work building, securing, and troubleshooting enterprise systems - Active Directory, SIEM deployment, networking, and virtualization. The kind of practical experience valued in security, cloud, and infrastructure roles.

The lab exists as an isolated, always-breakable environment for testing attacks, defenses, and misconfigurations without risking production or customer infrastructure. It takes concepts from CompTIA Security+ material, on-the-job field work, and day-to-day Linux usage and turns them into concrete scenarios that can be built, torn down, and improved over time.

Core goals include:

Converting theory into observable packets, logs, and alerts.
Building a “mini-enterprise” that behaves like the small networks seen in the field.
Practicing attack–defend cycles that replicate real-world workflows rather than one-off labs.
Evolution and expansion of tools and methodologies.
High-Level Design

The lab is implemented as a compact cyber range composed of virtual machines and, over time, additional physical hosts. The environment includes:

One or more hypervisors hosting domain-joined clients and servers.
A lightweight directory infrastructure (for example, an AD- or OpenLDAP-style domain).
Vulnerable services and deliberately misconfigured hosts for exploit and detection exercises.
Repurposed laptops operating as home servers.
Raspberry Pi systems providing DNS filtering, logging, and monitoring functions.
A segmented network design with: “User” subnet(s). “Server” subnet(s). “Management/monitoring” subnet where possible.
Within that layout, the lab supports:

Network scanning and enumeration (Nmap and similar tools).
Traffic capture and protocol analysis (e.g., Wireshark, tcpdump).
Centralized logging and basic SIEM-like workflows.
This makes it possible to test how poor network design, weak credentials, or exposed services could be discovered and exploited, and how to detect that activity from a blue-team perspective.

Usage and Workflow - The lab is meant to be noisy, experimental, and disposable. The standard workflow looks like:

Define a scenario, Example: “Misconfigured SMB share”
“Overly permissive firewall rule,” or “Unpatched web application.”
Deploy or reconfigure the relevant hosts and services.
Run reconnaissance and attacks using scanning, exploitation frameworks, or custom tooling.
Capture artifacts:

Packet captures.
Host and network logs.
Short notes on commands used and observations.
Apply mitigations and repeat the same attacks to confirm the controls work.
Snapshots, backups, and templates are used heavily so that entire sections of the environment can be destroyed, rebuilt, and iterated on without hesitation. The idea is to make breaking things and fixing them a normal, expected part of the process.

Documentation Approach - Each scenario gets recorded in a lightweight, technical format rather than a narrative blog post. For every exercise, the plan is to track:

Objective: what concept, control, or behavior is being validated.
Topology: which hosts, networks, and devices are in play.
Procedure: commands, tools, and configuration changes.
Results: what was observable, what failed, and what succeeded.
Hardening: final state, mitigations, and lessons to reuse later.
Over time, this builds a personal runbook of lab-tested configurations, troubleshooting steps, and incident-style notes that can feed directly into on-the-job problem solving, and future lab expansions.
