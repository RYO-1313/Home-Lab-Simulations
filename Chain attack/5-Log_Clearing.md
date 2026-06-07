# 🧹 5 — Defense Evasion: Log Clearing Detection with Wazuh + Splunk

> **Path:** Home-Lab-Simulations / Chain attack / 5-Log_Clearing

---

> ⬅️ Previous Simulation: [Chain attack — Persistence](../Persistence/)

---

## 🧠 What This Simulation Is About

An attacker who has moved through reconnaissance, initial access, privilege escalation, and persistence has one final priority before going quiet — **erasing the evidence**. Windows event logs contain a full record of everything that happened: every failed login, every privilege change, every new account created. Clearing them is the attacker's attempt to make the investigation harder.

In this simulation, I cleared both the Security and System logs on the compromised Windows machine from a remote Kali SSH session, then confirmed that Wazuh and Splunk caught the clearing event itself. The goal was to demonstrate a core defender's truth: **the act of deleting evidence is itself evidence.**

This maps directly to **MITRE ATT&CK T1070.001 — Indicator Removal: Clear Windows Event Logs**.

---

## 🖥️ Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| Attacker | Kali Linux (SSH into Windows) | — |
| Victim | Windows (DESKTOP-OGUH4L2) | 192.168.11.116 |
| SIEM Manager | Debian (Wazuh) | 192.168.11.132 |
| Log Aggregator | Splunk | — |

**Tools Used:** SSH · wevtutil · Wazuh Agent · Wazuh Manager · Splunk

---

## 🎯 SOC Relevance

Log clearing is one of the **highest-confidence indicators of malicious intent** an L1 analyst will ever see. Legitimate administrators almost never clear event logs — and when they do, it is a planned, documented, and authorized action.

When Windows Event IDs `1102` (Security log cleared) or `104` (System log cleared) appear in a Splunk alert, the correct response is immediate escalation. There is no benign explanation for these events appearing unexpectedly. This simulation trains the analyst to recognize them on sight and understand exactly what the attacker was trying to hide.

---

## 📊 Dashboard Before the Attack

<img width="1917" height="914" alt="Screenshot From 2026-06-03 15-13-57" src="https://github.com/user-attachments/assets/202cc0be-2bcc-4f62-a358-4261b07aa557" />


The dashboard reflects the accumulated alerts from the full attack chain — every event from the port scan through persistence is still visible. This is what the attacker wants to erase.

---

## 💥 Step 1 — Clear the Windows Event Logs

From **Kali**, via SSH into the Windows target, run the log clearing commands:

Clear the **Security log** — where authentication, privilege, and account events are stored:
```cmd
wevtutil cl Security
```

Clear the **System log** — where system-level events including service starts and stops are stored:
```cmd
wevtutil cl System
```

> `wevtutil cl` (clear log) is a built-in Windows utility. An attacker with admin access can run it silently in seconds, wiping the entire event history on the local machine. What they don't account for is that Wazuh has already forwarded those events to the SIEM — and that the clearing action itself generates new events.

---

## 🔎 Step 2 — Investigate in Splunk

In **Splunk Search and Reporting**, run:

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" rule.id="63103" | table _time, rule.description, data.win.logFileCleared.subjectUserName, rule.level | sort -_time
```

<img width="1917" height="892" alt="Screenshot From 2026-06-03 15-31-39" src="https://github.com/user-attachments/assets/a65fa2b6-f0f6-4607-8fe6-006f230cb9f2" />


### What the alert tells us

| Windows Event ID | Log | What It Means |
|-----------------|-----|---------------|
| `1102` | Security | The Security audit log was cleared |

| Field | Value |
|-------|-------|
| Rule Level | High |
| Triggered By | The `hacker` admin account (from 3-Privilege_Escalation) |
| Wazuh Rule | 63103 |
| MITRE Tag | T1070.001 — Indicator Removal |

<img width="1917" height="920" alt="Screenshot From 2026-06-03 15-33-33" src="https://github.com/user-attachments/assets/8affa99b-9bc9-425f-b260-96a3c2eb332c" />


> **The critical insight:** The attacker cleared the logs on the Windows machine — but those events had already been shipped to Wazuh and Splunk. The local machine is dark, but the SIEM has the full record. Furthermore, the clearing event itself (`1102` and `104`) is now in the SIEM, timestamped, with the username of whoever did it. The attacker erased evidence and left a new piece of evidence in the process.

---

## 📊 Dashboard After the Attack

<img width="1917" height="920" alt="Screenshot From 2026-06-03 15-33-33" src="https://github.com/user-attachments/assets/24bd9a2e-8eff-40f3-b9a4-2afbc584dc81" />


The dashboard shows new activity — the log clearing events are now the most recent alerts, sitting on top of the full kill chain we've been building since SIM_1.

---

## 🧩 MITRE ATT&CK Mapping

| Field | Value |
|-------|-------|
| Tactic | Defense Evasion |
| Technique | T1070.001 — Indicator Removal: Clear Windows Event Logs |
| Tool Used | wevtutil (native Windows utility) |
| Detection Method | Wazuh Windows Event ID 1102 / 104 monitoring → Splunk alert |
| Key Event IDs | 1102 (Security log cleared) · 104 (System log cleared) |

---

## 📋 Key Takeaways

- **Log clearing is self-defeating.** The attacker cleared the local logs to hide their tracks, but the SIEM already had everything. Forwarding logs off the endpoint in real time is one of the most important architectural decisions in a SOC — it means local tampering has zero impact on the investigation.
- **Event IDs 1102 and 104 are automatic escalation signals.** There is almost no legitimate reason for these events to appear unexpectedly. Any analyst who sees them in a Splunk dashboard should treat it as an active incident until proven otherwise. Memorize these two IDs.
- **This is the end of the kill chain — and the summary of it.** Every technique in this simulation series — reconnaissance, initial access, privilege escalation, persistence, and now evasion — follows the same attacker playbook. Seeing log clearing appear in the SIEM tells an analyst that whoever they're chasing has been operating with intent and method for a while. It means there's more to find.

---

## 🔗 Full Simulation Series

This simulation is the final stage of a complete attacker lifecycle lab:

| # | Simulation | MITRE Technique |
| 1 | [Port Scan Detection](1-Port_Scan.md) | T1046 — Network Service Discovery |
| 2 | [SSH Brute Force](2-SSH_BruteForce.md) | T1110.001 — Brute Force: Password Guessing |
| 3 | [Privilege Escalation](3-Privilege_Escalation.md) | T1078 / T1136 — Valid Accounts / Create Account |
| 4 | [Persistence](4-Persistence.md) | T1053.005 / T1547.001 — Scheduled Task / Registry Run Key |
| 5 | [Log Clearing](5-Log_Clearing.md) | T1070.001 — Indicator Removal |
---

*by [ryo](https://github.com/RYO-1313)*
