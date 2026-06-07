# 🏠 Home Lab Simulations

![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-Mapped-red?style=for-the-badge&logo=target&logoColor=white)
![Simulations](https://img.shields.io/badge/Simulations-5-blue?style=for-the-badge&logo=flask&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-Windows%20%7C%20Kali%20Linux-informational?style=for-the-badge&logo=linux&logoColor=white)
![SIEM](https://img.shields.io/badge/SIEM-Wazuh%20%2B%20Splunk-purple?style=for-the-badge&logo=elastic&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=for-the-badge)
![Focus](https://img.shields.io/badge/Focus-SOC%20%7C%20Blue%20Team-darkblue?style=for-the-badge&logo=shield&logoColor=white)

A hands-on cybersecurity lab where I simulate real attacker behavior end-to-end — from the first network probe to covering tracks — and detect every stage using industry-standard SIEM tools.

Each simulation is built in an isolated local environment using **Kali Linux**, **Windows**, **Wazuh**, and **Splunk**. Every technique is mapped to the **MITRE ATT&CK framework**, the same reference used by SOC teams worldwide to classify and respond to threats.

---

## 🎯 What This Lab Demonstrates

This isn't a collection of isolated exercises. The simulations in this repo follow a **complete attacker lifecycle** — the same sequence a real threat actor uses to infiltrate, control, and disappear inside a network.

As a defender, understanding each stage of that lifecycle is what makes the difference between catching an attacker early and finding out after the damage is done.

---

## 🔗 Current Simulations — The Full Kill Chain

| # | Simulation | MITRE Technique |
|---|-----------|----------------|
| 1 | [Port Scan Detection](1-Port_Scan.md) | T1046 — Network Service Discovery |
| 2 | [SSH Brute Force](2-SSH_BruteForce.md) | T1110.001 — Brute Force: Password Guessing |
| 3 | [Privilege Escalation](3-Privilege_Escalation.md) | T1078 / T1136 — Valid Accounts / Create Account |
| 4 | [Persistence](4-Persistence.md) | T1053.005 / T1547.001 — Scheduled Task / Registry Run Key |
| 5 | [Log Clearing](5-Log_Clearing.md) | T1070.001 — Indicator Removal |

> More simulations are actively in development and will be added to this repo.

---

## 🛠️ Tools & Technologies

`Kali Linux` · `Windows Server` · `Wazuh` · `Splunk` · `Nmap` · `Hydra` · `MITRE ATT&CK`

---

*by [ryo](https://github.com/RYO-1313)*
