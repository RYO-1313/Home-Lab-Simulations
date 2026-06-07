# 🏠 Home Lab Simulations

A personal cybersecurity home lab where I simulate real attacker techniques end-to-end — from the first reconnaissance probe to covering tracks — and practice detecting every stage using Wazuh and Splunk.

Each simulation is a standalone exercise mapped to the MITRE ATT&CK framework. Together, the first five form a complete attack chain: one attacker, one target, one full intrusion lifecycle.

---

## ⚔️ The Story So Far

A single attacker moves through a Windows target from first contact to evasion — and gets caught at every step.

| # | Simulation | What Happens | MITRE |
|---|-----------|-------------|-------|
| SIM_1 | [Port Scan Detection](./SIM_1_PortScan/) | Attacker maps the network, Wazuh catches the probe | T1046 |
| SIM_2 | [SSH Brute Force](./SIM_2_SSHBruteForce/) | Attacker forces their way in, Splunk catches the breach | T1110.001 |
| SIM_3 | [Privilege Escalation](./SIM_3_PrivilegeEscalation/) | Attacker climbs to admin, Event IDs tell the story | T1078 / T1136 |
| SIM_4 | [Persistence](./SIM_4_Persistence/) | Attacker plants two backdoors, both detected | T1053.005 / T1547.001 |
| SIM_5 | [Log Clearing](./SIM_5_LogClearing/) | Attacker tries to erase evidence — the SIEM already has it | T1070.001 |

More simulations coming as the lab grows.

---

## 🛠️ Lab Stack

- **Attacker:** Kali Linux
- **Victim:** Windows 10
- **SIEM:** Wazuh + Splunk
- **Network:** Isolated local lab

---

*by [ryo](https://github.com/RYO-1313)*
