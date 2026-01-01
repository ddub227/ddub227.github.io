---
layout: post
title: "My First Homelab Blog Post"
---

This document defines a personal cybersecurity lab built to connect existing experience in physical security, DMP systems, and Linux with the kind of hands-on work expected from SOC analysts and junior security engineers. The goal is not just to pass Security+, but to develop repeatable, real-world skills in monitoring, hardening, and breaking systems in a controlled environment.

Purpose of the Lab
The lab exists as an isolated, always-breakable environment for testing attacks, defenses, and misconfigurations without risking production or customer infrastructure. It takes concepts from Security+ material, on-the-job field work, and day-to-day Linux usage and turns them into concrete scenarios that can be built, torn down, and improved over time.

Core goals include:

Converting theory (ports, protocols, IAM, incident response) into observable packets, logs, and alerts.

Building a “mini-enterprise” that behaves like the small networks seen in the field.

Practicing attack–defend cycles until the workflows start to feel like normal operations rather than one-off labs.

High-Level Design
The environment is structured as a compact cyber range running on a mix of virtual machines and optionally a few physical devices:

One or more hypervisors hosting:

Domain-joined clients and servers.

A small directory environment (e.g., AD/OpenLDAP-style).

Vulnerable services and intentionally misconfigured hosts.

A segmented network design with:

“User” subnet(s).

“Server” subnet(s).

“Management/monitoring” subnet where possible.

Within that layout, the lab supports:

Network scanning and enumeration (Nmap and similar tools).

Traffic capture and protocol analysis (e.g., Wireshark, tcpdump).

Centralized logging and basic SIEM-like workflows.

Tying in Physical Security Experience
Field experience with alarms, access control, and DMP hardware is treated as part of the threat surface, not an afterthought. The lab is designed to mirror that intersection of physical and cyber:

Controllers and panels represented as:

Real hardware where available, or

Virtual stand-ins that expose similar services/ports.

Traffic between “field” devices, application servers, and management stations routed through the lab network, so it can be:

Monitored.

Intentionally weakened.

Probed and hardened.

This makes it possible to test how poor network design, weak credentials, or exposed services around physical security devices could be discovered and exploited, and how to detect that activity from a blue-team perspective.

Usage and Workflow
The lab is meant to be noisy, experimental, and disposable. The standard workflow looks like:

Define a scenario

Example: “Misconfigured SMB share,” “Overly permissive firewall rule,” or “Unpatched web application.”

Deploy or reconfigure the relevant hosts and services.

Run reconnaissance and attacks using scanning, exploitation frameworks, or custom tooling.

Capture artifacts:

Packet captures.

Host and network logs.

Short notes on commands used and observations.

Apply mitigations and repeat the same attacks to confirm the controls work.

Snapshots, backups, and templates are used heavily so that entire sections of the environment can be destroyed, rebuilt, and iterated on without hesitation. The idea is to make breaking things and fixing them a normal, expected part of the process.

Documentation Approach
Each scenario gets recorded in a lightweight, technical format rather than a narrative blog post. For every exercise, the plan is to track:

Objective: what concept, control, or behavior is being validated.

Topology: which hosts, networks, and devices are in play.

Procedure: commands, tools, and configuration changes.

Results: what was observable, what failed, and what succeeded.

Hardening: final state, mitigations, and lessons to reuse later.

Over time, this builds a personal runbook of lab-tested configurations, troubleshooting steps, and incident-style notes that can feed directly into interviews, on-the-job problem solving, and future lab expansions.
