# Home Lab Simulations

![Security](https://img.shields.io/badge/Category-Attack%20Simulations-red?style=flat-square)
![Focus](https://img.shields.io/badge/Current%20Focus-Brute%20Force-orange?style=flat-square)
![Documentation](https://img.shields.io/badge/Documentation-NIST%20%7C%20SANS-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-green?style=flat-square)

<br>

## About

I am a self-taught cybersecurity student with a clear goal — break into the field as a SOC Analyst and use that as the foundation for a long-term career in offensive security. Every simulation in this repository is a deliberate step toward that goal: building real detection experience, practicing incident response, and understanding attacks from both sides.

The lab environment powering these simulations is fully documented, configured, and maintained in a separate repository: [home-soc-lab](https://github.com/RYO-1313/home-soc-lab). It runs Splunk as a SIEM and Wazuh as an EDR on a Debian 13 machine, connected to Windows and Kali Linux endpoints over a local bridged network.

<br>

---

## Purpose

This repository documents controlled attack simulations performed in an isolated local network. The goal is not just to run attacks — it is to observe how they look from a defender's perspective, practice triage and incident response, and produce professional documentation that reflects real SOC workflows.

Each simulation is broken into three sides:

**Victim Side** — target machine configuration, exposed services, and security posture before the attack.

**Attacker Side** — tools used, commands executed, and the technique being simulated.

**Manager Side** — how Wazuh and Splunk detected the attack, what the alerts looked like, and the investigation process inside the SIEM.

<br>

---

## Current Focus — Brute Force Attacks

The repository is currently focused on brute force and credential attacks. This covers SSH brute force using Hydra, Windows logon failures, detection patterns in Wazuh, and how those alerts surface inside Splunk dashboards.

Future simulations will expand into other attack categories as the lab grows.

<br>

---

## Simulations

| ID | Attack Type | Target | Tool | Status |
|----|------------|--------|------|--------|
| [INC-001](Documents/INC-001_SSH_BruteForce.pdf) | SSH Brute Force | Windows (DESKTOP-OGUH4L2) | Hydra v9.6 | ✅ Complete |

<br>

---

## Documentation Template

Every simulation in this repository is documented using a structured incident report based on:

- **NIST SP 800-61 Rev.3** — Computer Security Incident Handling Guide
- **SANS Incident Handler's Handbook** — Incident response phases and best practices

Reports cover incident identification, attack timeline, indicators of compromise, MITRE ATT&CK mapping, triage decision, containment actions, lessons learned, and a full analyst narrative.

<br>

---

> This is an active, growing repository. Simulations and documentation are added as the lab expands.
