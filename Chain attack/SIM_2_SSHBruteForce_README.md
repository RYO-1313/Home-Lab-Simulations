# 🔐 SSH Brute Force Detection with Wazuh + Splunk

> **Path:** Home-Lab-Simulations / Chain attack / SSH-BruteForce

---

> ⬅️ Previous Simulation: [Chain attack — Port Scan Detection](../PortScan/)
> ➡️ Next Simulation: [Chain attack — Privilege Escalation](../PrivilegeEscalation/)

---

## 🧠 What This Simulation Is About

After mapping the network, an attacker's next move is often to force their way in. **SSH brute forcing** is one of the most common techniques — systematically trying passwords until one works.

In this simulation, I launched a live dictionary attack using Hydra from Kali Linux against a Windows SSH server, then tracked every step of the breach through Wazuh and Splunk. The goal was to confirm the full detection pipeline: from the first failed login attempt, to the moment the attacker gets in, to the post-compromise privilege escalation alert.

This maps directly to **MITRE ATT&CK T1110.001 — Brute Force: Password Guessing**.

> **Note:** This is a controlled lab simulation performed on an isolated local network. All machines are owned and operated by the analyst. Never perform this against systems you do not own.

---

## 🖥️ Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| Attacker | Kali Linux | — |
| Victim | Windows (DESKTOP-OGUH4L2) | 192.168.11.116 |
| SIEM Manager | Debian (Wazuh) | 192.168.11.132 |
| Log Aggregator | Splunk | — |

**Tools Used:** Hydra · OpenSSH · Wazuh Agent · Wazuh Manager · Splunk

---

## 🎯 SOC Relevance

SSH brute force attacks are among the **most common threats in any SOC alert queue**. Automated bots scan the internet constantly looking for exposed SSH ports, and once found, they hammer them with credential lists.

The detection pattern here is textbook: a flood of authentication failures from a single source, followed immediately by a successful login. That transition — failure → success — is the critical moment an analyst must catch. Missing it means the attacker is already inside. This simulation trains that exact recognition skill against a live attack.

---

## ✅ Step 0 — Verify Connectivity

Before anything else, confirm Kali can reach the Windows target.

**On Kali:**
```bash
ping 192.168.11.116
```

📸 `[SCREENSHOT — Ping output showing 0% packet loss]`

6 packets transmitted, 6 received, 0% packet loss — machines can communicate.

---

## ⚙️ Step 1 — Enable OpenSSH Server on Windows

Hydra needs an SSH server running on the target. Windows doesn't enable this by default.

**Option A — Using Windows Settings:**

1. Press the **Windows key** and open **Settings**
2. Search for **Add an Optional Feature**
3. Click **Add a feature** or **View Features**
4. Search for **OpenSSH Server**
5. Click **Install**

**Option B — Using CMD as Administrator:**

Install OpenSSH Server:
```cmd
dism /online /Add-Capability /CapabilityName:OpenSSH.Server~~~~0.0.1.0
```

Start the SSH service:
```cmd
net start sshd
```

Verify SSH is listening on port 22:
```cmd
netstat -an | findstr 22
```

📸 `[SCREENSHOT — netstat output confirming port 22 LISTENING]`

Confirm the Windows IP:
```cmd
ipconfig
```

📸 `[SCREENSHOT — ipconfig output showing 192.168.11.116]`

---

## ⚙️ Step 2 — Build the Password List

On **Kali**, create a custom wordlist with the real password buried at the end:

```bash
nano passwords.txt
```

Add the following:

```
batman2007
shadow99
letmein2007
qwerty123
dragon2007
pass1234
monkey07
sunshine2007
master99
football2007
welcome1
princess2007
abc12345
mustang07
starwars2007
hunter99
baseball2007
iloveyou
charlie2007
2007
```

Save with `Ctrl+X` then `Enter`.

> The real password (`2007`) is placed last — Hydra will cycle through every entry before hitting it, generating the full flood of failures we want Wazuh to catch.

---

## 💥 Step 3 — Launch the Brute Force Attack

On **Kali**, run Hydra:

```bash
hydra -l HAL -P passwords.txt ssh://192.168.11.116 -t 4 -v
```

| Flag | What it does |
|------|-------------|
| `-l HAL` | Target username |
| `-P passwords.txt` | Wordlist file containing candidate passwords |
| `ssh://192.168.11.116` | Target protocol and IP |
| `-t 4` | 4 parallel threads |
| `-v` | Verbose output — shows each attempt in real time |

📸 `[SCREENSHOT — Hydra output showing the attack in progress and cracked credentials]`

Hydra completed the attack in **8 seconds**, finding the correct credentials:

- **Username:** `HAL`
- **Password:** `2007`

---

## 📊 Step 4 — Observe the Attack in Splunk

### Dashboard before the attack

📸 `[SCREENSHOT — Splunk dashboard before attack, 78 total alerts, flat timeline]`

Before the attack, the dashboard showed **78 total alerts** with a flat, quiet timeline — normal background noise.

### Dashboard after the attack

📸 `[SCREENSHOT — Splunk dashboard after attack, 297 total alerts, visible spike]`

After the attack, the total jumped to **297 alerts** with a clear spike on the timeline — a sudden burst of abnormal activity that no analyst could miss.

---

## 🔎 Step 5 — Investigate the Alert in Splunk

In **Splunk Search and Reporting**, run:

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" | stats count by rule.description | sort -count
```

📸 `[SCREENSHOT — Splunk search results showing alert breakdown by rule description]`

### What the alerts tell us

**Attack phase — Hydra cycling through the wordlist:**

| Alert | Count | What It Means |
|-------|-------|---------------|
| Windows audit failure event | 40 | Windows logging every failed authentication attempt |
| Logon Failure — Unknown user or bad password | 40 | Hydra working through the wordlist one entry at a time |

**Breach phase — attacker succeeded:**

| Alert | Count | What It Means |
|-------|-------|---------------|
| Windows Logon Success | 12 | Hydra found the correct password — breach confirmed |
| Special privileges assigned to new logon | 30 | Elevated privileges were granted post-compromise |

📸 `[SCREENSHOT — Splunk alert detail showing logon success appearing after the flood of failures]`

> **The critical signal:** A `Windows Logon Success` event appearing immediately after a flood of `Logon Failure` events from the same source. That sequence means one thing — the attacker is in. An L1 analyst seeing this pattern should treat it as an active incident and escalate immediately.

---

## 🧩 MITRE ATT&CK Mapping

| Field | Value |
|-------|-------|
| Tactic | Credential Access |
| Technique | T1110.001 — Brute Force: Password Guessing |
| Tool Used | Hydra |
| Detection Method | Wazuh failed login correlation → Splunk alert spike |
| Key Signal | Logon Success immediately following flood of Logon Failures |

---

## 📋 Key Takeaways

- **Volume alone isn't the alert — the transition is.** 40 failures is suspicious. 40 failures followed by a success is an active breach. Training your eye to spot that pattern in a Splunk timeline is one of the most important skills an L1 analyst can develop.
- **8 seconds to compromise.** A 20-entry wordlist cracked in under 10 seconds shows how fast brute force works against weak credentials. This reinforces why failed login thresholds and account lockout policies are critical defensive controls.
- **Wazuh catches the full story.** From the first failed attempt to the privilege assignment post-login, the detection pipeline captured every stage of the attack. A complete alert chain like this is what enables proper incident reporting — you're not guessing what happened, you're reading it.

---

*by [ryo](https://github.com/RYO-1313)*
