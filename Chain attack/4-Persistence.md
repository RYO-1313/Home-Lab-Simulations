# 🪝 4 — Persistence Detection with Wazuh + Splunk

> **Path:** Home-Lab-Simulations / Chain attack / Persistence

---

> ⬅️ Previous Simulation: [Chain attack — Privilege Escalation](../PrivilegeEscalation/)
> ➡️ Next Simulation: [Chain attack — Defense Evasion: Log Clearing](../LogClearing/)

---

## 🧠 What This Simulation Is About

After an attacker gets in and escalates privileges, their biggest fear is losing access. Sessions close, victims reboot, and IT teams find and delete compromised accounts. **Persistence** is the attacker's answer to that problem — planting mechanisms that survive reboots, account deletions, and standard cleanup, guaranteeing a way back in without needing to break in again.

In this simulation, I planted two persistence mechanisms on the compromised Windows machine while operating as the `hacker` admin account created in 3-Privilege Escalation: a **Scheduled Task** disguised as a Windows system process, and a **Registry Run Key** that executes on every system startup. Both were detected by Wazuh and surfaced in Splunk.

This maps directly to **MITRE ATT&CK T1053.005 — Scheduled Task** and **T1547.001 — Registry Run Keys / Startup Folder**.

> **Note:** This is a controlled lab simulation performed on an isolated local network. All machines are owned and operated by the analyst. Never perform this against systems you do not own.

---

## 🖥️ Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| Attacker | Kali Linux (SSH into Windows) | — |
| Victim | Windows (DESKTOP-OGUH4L2) | 192.168.11.116 |
| SIEM Manager | Debian (Wazuh) | 192.168.11.132 |
| Log Aggregator | Splunk | — |

**Tools Used:** SSH · Windows CMD · schtasks · reg · Wazuh Agent · Wazuh Manager · Splunk · Windows Task Scheduler · Windows Registry

---

## 🎯 SOC Relevance

Persistence is what turns a one-time intrusion into a long-term compromise. Attackers who establish persistence can return at will — days, weeks, or months after the initial breach — making it critical for SOC analysts to detect these mechanisms early.

The two techniques simulated here — scheduled tasks and registry run keys — are among the **most frequently abused persistence methods** in real-world malware and APT campaigns. Learning to spot them in Wazuh alerts and Splunk searches is a direct, practical skill for any L1 analyst working endpoint detection.

---

## ⚙️ Step 0 — Configure Wazuh for Task Scheduler Logging

By default, Windows Task Scheduler events are not forwarded. We enable them before the attack.

**On Windows, open CMD as Administrator:**

Enable auditing for object access events:
```cmd
auditpol /set /subcategory:"Other Object Access Events" /success:enable /failure:enable
```

Enable the Task Scheduler operational log:
```cmd
wevtutil set-log "Microsoft-Windows-TaskScheduler/Operational" /enabled:true
```

📸 `[SCREENSHOT — CMD output confirming both commands executed successfully]`

---

## 📊 Dashboard Before the Attack

📸 `[SCREENSHOT — Splunk dashboard showing baseline alert count before any persistence is planted]`

The dashboard is quiet — no persistence-related alerts. This is our clean baseline.

---

## 💥 Technique 1 — Scheduled Task

### What It Is

A **Scheduled Task** is a built-in Windows feature that runs programs automatically at set times or system events. Attackers abuse it by creating tasks that execute malicious payloads on startup, disguising them with names that look like legitimate Windows processes.

### Step 1 — Confirm Admin Access

Before planting persistence, verify the current user context on the Windows machine:

```cmd
whoami
```

📸 `[SCREENSHOT — whoami output confirming we're running as the hacker admin account]`

### Step 2 — Create the Malicious Scheduled Task

Create a task named `WindowsUpdateService` — a name designed to blend in with legitimate Windows maintenance tasks:

```cmd
schtasks /create /tn "WindowsUpdateService" /tr "cmd.exe /c echo persistence > C:\Windows\Temp\backdoor.txt" /sc onstart /ru SYSTEM
```

| Flag | What It Does |
|------|-------------|
| `/tn "WindowsUpdateService"` | Task name — disguised as a Windows system process |
| `/tr "cmd.exe /c ..."` | The command to run — writes a file to disk as proof of execution |
| `/sc onstart` | Trigger — runs every time the system starts |
| `/ru SYSTEM` | Runs as the SYSTEM account — the highest privilege level on Windows |

Verify the task was created:
```cmd
schtasks /query /tn "WindowsUpdateService"
```

📸 `[SCREENSHOT — schtasks query output confirming the task exists and its schedule]`

### Step 3 — Detect in Splunk

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" rule.id="60228" | table _time, rule.description, data.win.eventdata.subjectUserName, data.win.eventdata.taskName, rule.level | sort -_time
```

📸 `[SCREENSHOT — Splunk results showing the scheduled task creation alert]`

> **What to look for:** A new scheduled task created by a non-system user, with a name mimicking a Windows process, set to run at SYSTEM privilege on startup. That combination is a persistence flag. Legitimate IT tasks are typically deployed through Group Policy, not manually via CMD.

---

## 💥 Technique 2 — Registry Run Key

### What It Is

The **Registry Run Key** is one of the oldest and most reliable persistence mechanisms in existence. Any value placed under `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` will execute automatically every time Windows starts — no user interaction required.

### Step 1 — Plant the Registry Key

From **Kali**, via SSH into the Windows target, run:

```bash
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v "WindowsSecurityUpdate" /t REG_SZ /d "cmd.exe /c echo persistence > C:\Windows\Temp\backdoor2.txt" /f
```

| Part | What It Does |
|------|-------------|
| `HKLM\...\Run` | Registry location — executes on every system startup |
| `/v "WindowsSecurityUpdate"` | Key name — disguised as a Windows security process |
| `/t REG_SZ` | Data type — string value |
| `/d "cmd.exe /c ..."` | The command that runs on startup |
| `/f` | Force — overwrites without prompting |

Verify the key was written:
```bash
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

📸 `[SCREENSHOT — reg query output confirming WindowsSecurityUpdate is present in the Run key]`

### Step 2 — Detect in Splunk

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" | search "WindowsSecurityUpdate" | table _time, rule.description, syscheck.path, syscheck.value_name, rule.level
```

📸 `[SCREENSHOT — Splunk results showing the registry modification alert]`

> **What to look for:** A modification to `HKLM\...\CurrentVersion\Run` by a non-administrative process or an unexpected user account. Wazuh's FIM (File Integrity Monitoring) catches registry changes in real time. A new value appearing here outside of a known software installation is a persistence indicator that warrants immediate investigation.

---

## 📊 Dashboard After the Attack

📸 `[SCREENSHOT — Splunk dashboard after both persistence mechanisms are planted, showing new alerts]`

Two new alert categories have appeared — one for the scheduled task creation, one for the registry modification. Both are now on the analyst's radar.

---

## 🧩 MITRE ATT&CK Mapping

| Technique | ID | Method | Detection |
|-----------|-----|--------|-----------|
| Scheduled Task/Job | T1053.005 | schtasks /create, SYSTEM privilege, startup trigger | Wazuh rule 60228 → Splunk |
| Boot/Logon Autostart — Registry Run Keys | T1547.001 | HKLM\...\CurrentVersion\Run registry modification | Wazuh FIM → Splunk |

---

## 📋 Key Takeaways

- **Naming matters in evasion.** Both persistence mechanisms used names designed to look legitimate — `WindowsUpdateService` and `WindowsSecurityUpdate`. Attackers rely on analysts glossing over familiar-looking names. Scrutinizing task names and run key values — especially ones that don't match known installed software — is a critical habit.
- **Persistence is planted quietly.** Neither technique requires user interaction or produces obvious visual output on the victim machine. The only evidence is in the logs — which is exactly why SIEM coverage of scheduled tasks and registry modifications is non-negotiable.
- **Two techniques, two detection paths.** Task creation is caught via Windows Event ID monitoring (Wazuh rule 60228). Registry changes are caught via Wazuh's File Integrity Monitoring on the registry. Knowing which detection mechanism covers which technique helps an analyst understand gaps in coverage — and where an attacker might try to hide.

---

*by [ryo](https://github.com/RYO-1313)*
