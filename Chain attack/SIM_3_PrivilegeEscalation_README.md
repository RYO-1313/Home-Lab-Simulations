# ⬆️ SIM_3 — Privilege Escalation Detection with Wazuh + Splunk

> **Path:** Home-Lab-Simulations / SIM_3 / PrivilegeEscalation

---

> ⬅️ Previous Simulation: [SIM_2 — SSH Brute Force](../SSH-BruteForce/)
> ➡️ Next Simulation: [SIM_4 — Persistence](../Persistence/)

---

## 🧠 What This Simulation Is About

Getting into a system is only half the battle. Most compromised accounts start with limited access — not enough to install software, create users, or move laterally across the network. **Privilege escalation** is how an attacker climbs from a low-privilege foothold to full administrative control.

In this simulation, I created a low-privilege user (`victim`) on the Windows target, SSH'd in from Kali to simulate a limited compromise, then attempted and executed privilege escalation to the Administrators group. Wazuh and Splunk captured both the failed attempt and the successful escalation. The goal was to demonstrate what escalation activity looks like in a SIEM — and why it always leaves a trace.

This maps directly to **MITRE ATT&CK T1078 — Valid Accounts** and **T1136 — Create Account**.

> **Note:** This is a controlled lab simulation performed on an isolated local network. All machines are owned and operated by the analyst. Never perform this against systems you do not own.

---

## 🖥️ Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| Attacker | Kali Linux | — |
| Victim | Windows (DESKTOP-OGUH4L2) | 192.168.11.116 |
| SIEM Manager | Debian (Wazuh) | 192.168.11.132 |
| Log Aggregator | Splunk | — |

**Tools Used:** SSH · Windows CMD · PowerShell · Wazuh Agent · Wazuh Manager · Splunk

---

## 🎯 SOC Relevance

Privilege escalation is one of the **most critical moments in any intrusion**. An attacker with user-level access is contained. An attacker with admin access can disable security tools, create backdoors, dump credentials, and move to other systems.

From a SOC perspective, the Windows Event IDs generated during escalation — `4720` (user created), `4732` (member added to privileged group) — are high-fidelity signals. They are rarely triggered by legitimate activity at odd hours or from unexpected accounts, which makes them reliable escalation indicators. This simulation trains the analyst to recognize those events and understand what they mean in context.

---

## ✅ Step 0 — Create a Low-Privilege User

To simulate a realistic compromise, we need an account with no administrative rights. We start by creating one directly on the Windows machine.

**On Windows, open CMD as Administrator:**

Create the user and a restricted group:
```cmd
net user victim Password123 /add
net localgroup analysts /add
net localgroup analysts victim /add
```

Verify the account was created with limited privileges:
```cmd
net user victim
```

📸 `[SCREENSHOT — net user output confirming victim account with no admin rights]`

> The `victim` account is intentionally weak — it belongs only to the `analysts` group with no path to administrative resources. This is the starting state of the simulated attacker's access.

---

## ⚙️ Step 1 — Log In as the Low-Privilege User

From **Kali**, SSH into the Windows target as `victim`:

```bash
ssh victim@192.168.11.116
```

Password: `Password123`

📸 `[SCREENSHOT — Successful SSH login as victim]`

Confirm who we are logged in as:
```bash
whoami
```

Check current group memberships:
```bash
whoami /groups
```

📸 `[SCREENSHOT — whoami /groups output showing victim's limited group membership]`

> At this point, the attacker is inside the machine but operating with minimal permissions. The target is `BUILTIN\Administrators` — the group that grants full control. `Mandatory Label\High Mandatory Level` is the marker of admin territory. Neither is present yet.

---

## 🔍 Step 2 — Test the Access Boundary

Before attempting escalation, we confirm the access restrictions are real.

**From the SSH session**, navigate to the admin user's desktop:
```cmd
cd "C:\Users\HAL\Desktop"
```

Attempt to create a file (this will fail):
```cmd
net user hacker Password123 /add
```

📸 `[SCREENSHOT — Access denied error when attempting to create a user as victim]`

> The `Access Denied` error is the access boundary working as intended. It also generates a Windows audit failure event that Wazuh will catch. This failed attempt is part of the evidence trail.

---

## 💥 Step 3 — Attempt Privilege Escalation via PowerShell

**Still in the SSH session**, attempt to add `victim` to the Administrators group using PowerShell:

```cmd
powershell -command "Add-LocalGroupMember -Group Administrators -Member victim"
```

> This command attempts to elevate `victim` directly into the Administrators group. On a properly hardened system this should fail without admin credentials — but the attempt itself is logged.

📸 `[SCREENSHOT — Splunk alert triggered by the escalation attempt]`

Wazuh catches this event and forwards it to Splunk. The alert fires even on a failed attempt — any account trying to add itself to a privileged group is suspicious regardless of success.

---

## 💥 Step 4 — Execute Successful Privilege Escalation

For the second stage of this simulation, escalation is performed using the admin account (`HAL`) to show what a successful escalation event looks like in the SIEM.

> In a real attack, this would happen if the attacker had already obtained the admin credentials — for example, through the brute force in SIM_2.

**On the Windows machine, using CMD as HAL (admin):**

Create a new privileged backdoor account:
```cmd
net user hacker Password123 /add
net localgroup Administrators hacker /add
```

Verify the account was created and added to Administrators:
```cmd
net user hacker
```

📸 `[SCREENSHOT — net user hacker output confirming Administrators group membership]`

---

## 📊 Step 5 — Investigate in Splunk

In **Splunk Search and Reporting**, run:

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" | search "4720" OR "4732" OR "hacker" | table _time, rule.description, data.win.eventdata.subjectUserName, data.win.eventdata.targetUserName, rule.level | sort -_time
```

📸 `[SCREENSHOT — Splunk results showing the escalation event chain]`

### What the alerts tell us

| Windows Event ID | What It Means |
|-----------------|---------------|
| `4720` | A new user account was created (`hacker`) |
| `4732` | A member was added to a security-enabled local group (Administrators) |

| Alert | Triggered By | Significance |
|-------|-------------|--------------|
| User Account Created | `HAL` created `hacker` | New account creation by a compromised admin is a red flag |
| Administrators Group Changed | `hacker` added to Administrators | Privilege escalation confirmed — attacker now has full control |

📸 `[SCREENSHOT — Alert detail showing HAL created hacker and added it to Administrators]`

> **The key signal:** Event `4720` followed by `4732` in rapid succession, triggered by the same user, is the textbook signature of an attacker creating a persistent admin backdoor. These two events together tell the complete story — a new account was created and immediately elevated.

---

## 🧩 MITRE ATT&CK Mapping

| Field | Value |
|-------|-------|
| Tactic | Privilege Escalation / Persistence |
| Technique | T1078 — Valid Accounts |
| Sub-technique | T1136 — Create Account |
| Detection Method | Wazuh Windows Event ID monitoring → Splunk correlation |
| Key Event IDs | 4720 (account created) · 4732 (added to privileged group) |

---

## 📋 Key Takeaways

- **Failed attempts are just as important as successful ones.** The `Access Denied` error and the failed PowerShell escalation both generated Wazuh alerts. In a real investigation, a chain of failed privilege attempts followed by success tells a complete story of an attacker probing defenses before breaking through.
- **Event IDs 4720 and 4732 are tier-1 escalation signals.** New account creation combined with immediate addition to a privileged group — especially outside business hours or from an unexpected account — should trigger immediate investigation. Knowing these event IDs by heart is table stakes for an L1 analyst.
- **The attacker's goal is always Administrators.** Every privilege escalation technique, regardless of method, is trying to reach the same destination. Understanding that goal helps an analyst prioritize — any activity touching the Administrators group deserves scrutiny, always.

---

*by [ryo](https://github.com/RYO-1313)*
