# SSH Brute Force Attack — Lab Simulation

This document walks through a controlled SSH brute force attack simulation using Hydra on Kali Linux against a Windows target. The goal is to observe how Wazuh EDR detects the attack and how the alerts appear in Splunk SIEM.

> **Note:** This is a controlled lab simulation performed on an isolated local network. All machines are owned and operated by the analyst. Never perform this against systems you do not own.

<br>

---

## Lab Environment

| Machine | OS | IP Address | Role |
|--------|-----|------------|------|
| SEIM | Debian 13 | 192.168.11.132 | Wazuh Manager + Splunk |
| DESKTOP-OGUH4L2 | Windows | 192.168.11.116 | Victim / Target |
| Kali Linux | Kali | 192.168.11.134 | Attacker |

<br>

---

## Step 1 — Enable OpenSSH Server on Windows (Victim Setup)

The Windows machine needs to be running an SSH server so Hydra has something to attack.

**Option A — Using Windows Settings:**

1. Press the **Windows key** and open **Settings**
2. Search for **Add an Optional Feature**
3. Click **Add a feature** or **View Features**
4. Search for **OpenSSH Server**
5. Click **Install**

**Option B — Using CMD as Administrator:**

```cmd
net start sshd
```

Verify SSH is listening on port 22:

```cmd
netstat -an | findstr 22
```

You should see port 22 in the output with a `LISTENING` state.

<br>

### Get the Windows IP address

In CMD as Administrator:

```cmd
ipconfig
```

<!-- Screenshot: Windows ipconfig output showing IP 192.168.11.116 -->

<br>

---

## Step 2 — Verify Network Connectivity from Kali

Before running the attack, confirm the Kali machine can reach the Windows target.

On Kali, run:

```bash
ping 192.168.11.116
```

<img width="560" height="305" alt="Screenshot From 2026-05-11 18-34-01" src="https://github.com/user-attachments/assets/a416a492-5c68-412d-9989-ffedc86e0f70" />


6 packets transmitted, 6 received, 0% packet loss confirms the machines can communicate.

<br>

---

## Step 3 — Run the Brute Force Attack with Hydra

On Kali Linux, run the following command:

```bash
hydra -l HAL -P passwords.txt ssh://192.168.11.116 -t 4 -v
```

| Flag | What it does |
|------|-------------|
| `-l HAL` | Target username |
| `-P passwords.txt` | Wordlist file containing candidate passwords |
| `ssh://192.168.11.116` | Target protocol and IP |
| `-t 4` | 4 parallel threads |
| `-v` | Verbose output |



Hydra v9.6 completed the attack in **8 seconds** (13:34:33 → 13:34:41), finding the correct credentials:

- **Username:** HAL
- **Password:** 2007

<br>

---

## Step 4 — Observe the Attack in Splunk

### Dashboard before the attack

<img width="1912" height="863" alt="Screenshot From 2026-05-11 18-22-22" src="https://github.com/user-attachments/assets/bb9f1524-b112-4efa-ae0e-4fd69cf1e0ab" />


Before the attack the dashboard showed **78 total alerts** with a flat, quiet timeline.

<br>

### Dashboard after the attack

<img width="1912" height="863" alt="Screenshot From 2026-05-11 18-24-24" src="https://github.com/user-attachments/assets/1ab4ed33-1725-476c-953d-a8a9b49c3bcd" />


After the attack the total jumped to **297 alerts** with a visible spike on the timeline — a clear sign of sudden abnormal activity.

<br>

### Investigate using Splunk search

Run the following search in Splunk Search and Reporting:

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" | stats count by rule.description | sort -count
```

<img width="1912" height="861" alt="Screenshot From 2026-05-11 18-25-12" src="https://github.com/user-attachments/assets/8018f22f-1cf0-4300-95df-145817a36215" />


<br>

### What the alerts tell us

**Attack phase — Hydra trying passwords:**

| Alert | Count | Meaning |
|-------|-------|---------|
| Windows audit failure event | 90 | Windows logging every failed authentication attempt |
| Logon Failure - Unknown user or bad password | 39 | Hydra cycling through the password wordlist |
| Multiple Windows Logon Failures | — | Wazuh pattern detection — brute force confirmed |

**Breach phase — attacker succeeded:**

| Alert | Count | Meaning |
|-------|-------|---------|
| Windows Logon Success | 38 | Hydra found the correct password — breach confirmed |
| Special privileges assigned to new logon | 52 | Elevated privileges granted post-compromise |
| Non service account logged off | 42 | Active sessions established on victim machine |

<img width="1912" height="861" alt="Screenshot From 2026-05-11 18-27-07" src="https://github.com/user-attachments/assets/803e3a6d-95ff-4303-94e0-a3da4ec36905" />


<br>

---

## What this simulation demonstrates

This lab confirms the full detection pipeline works as intended:

1. Hydra launches a dictionary attack against SSH on port 22
2. Wazuh EDR detects the failed login pattern in real time and generates alerts
3. The Splunk Universal Forwarder ships all alerts to the `wazuh-alerts` index
4. Splunk dashboards make the attack immediately visible — both the failed attempts and the successful breach

The most critical alert is **Windows Logon Success** appearing after a flood of failures. That is the signal that the attacker succeeded and an incident response must begin.

<br>

---

## Incident Report

The full incident report for this simulation is documented in:

[INC-001 — SSH Brute Force Incident Report](/Documents /INC-001_SSH_BruteForce.pdf)

<br>

---

> Last updated: May 2026 &nbsp;·&nbsp; Links reviewed every 2 months
