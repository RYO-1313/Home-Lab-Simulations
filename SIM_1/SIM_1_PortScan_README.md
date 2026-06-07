# 🔍 SIM_1 — Port Scan Detection with Wazuh + Splunk

> **Path:** Home-Lab-Simulations / SIM_1 / PortScan

---

> ➡️ Next Simulation: [SIM_2 — SSH Brute Force](../SSH-BruteForce/)

---

## 🧠 What This Simulation Is About

Before an attacker launches any real attack, they need a map. **Port scanning** is how they build it — probing a target system to discover which ports are open, which services are running, and where the weaknesses might be.

In this simulation, I launched a live Nmap SYN scan from Kali Linux against a Windows target, configured Wazuh to detect and forward firewall drop events, and tracked the scan activity through Splunk. The goal was to catch the earliest possible stage of an attack — the moment the attacker is still just looking.

This maps directly to **MITRE ATT&CK T1046 — Network Service Discovery**.

---

## 🖥️ Lab Environment

| Role | Machine | IP |
|------|---------|-----|
| Attacker | Kali Linux | — |
| Victim | Windows (DESKTOP-OGUH4L2) | 192.168.11.116 |
| SIEM Manager | Debian (Wazuh) | 192.168.11.132 |
| Log Aggregator | Splunk | — |

**Tools Used:** Nmap · Wazuh Agent · Wazuh Manager · Splunk · Windows Firewall · netsh

---

## 🎯 SOC Relevance

Port scans are **the first thing an attacker does and the first thing a SOC analyst should catch.** Detecting a scan early gives the blue team a critical window — the attacker hasn't compromised anything yet, and blocking them here stops the entire kill chain before it starts.

In a real SOC environment, a sudden spike in firewall DROP events from a single source IP is a tier-1 triage signal. This simulation trains exactly that skill: recognizing the pattern, correlating the source, and escalating before damage is done.

---

## ✅ Step 0 — Verify Connectivity

Before anything else, confirm all three machines can communicate.

**On Windows — ping the Wazuh Manager:**
```cmd
ping 192.168.11.132
```

**On Kali — ping the Windows target:**
```bash
ping 192.168.11.116
```

**On Wazuh Manager — ping the Windows target:**
```bash
ping 192.168.11.116
```


All machines are reachable. The lab is ready.

---

## ⚙️ Step 1 — Enable Windows Firewall Logging

By default, Windows does not log dropped connections. We need to enable that before Wazuh can pick anything up.

**On Windows, open CMD as Administrator:**

Enable logging for dropped connections:
```cmd
netsh advfirewall set allprofiles logging droppedconnections enable
```

Set the log file path:
```cmd
netsh advfirewall set allprofiles logging filename "%systemroot%\system32\LogFiles\Firewall\pfirewall.log"
```

Set max log file size:
```cmd
netsh advfirewall set allprofiles logging maxfilesize 4096
```

Make sure the firewall is on:
```cmd
netsh advfirewall set allprofiles state on
```

Verify the current state:
```cmd
netsh advfirewall show currentprofile | findstr "State"
```

📸 `[SCREENSHOT — Output confirming firewall State is ON and logging is enabled]`

---

## ⚙️ Step 2 — Configure the Wazuh Agent

Now we tell the Wazuh agent to watch the firewall log file we just enabled.

**On Windows, open CMD as Administrator:**

```cmd
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Add this block inside the `<ossec_config>` section:

```xml
<localfile>
  <location>C:\Windows\System32\LogFiles\Firewall\pfirewall.log</location>
  <log_format>syslog</log_format>
</localfile>
```

Save and close, then restart the agent to apply the change:

```cmd
net stop WazuhSvc && net start WazuhSvc
```

📸 `[SCREENSHOT — ossec.conf showing the new localfile block]`

---

## ⚙️ Step 3 — Configure Wazuh Detection Rules

By default, Wazuh doesn't know what a port scan looks like in a Windows firewall log. We write a custom rule to teach it.

**On the Wazuh Manager (Debian), open the local rules file:**

```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

Add the following rule group:

```xml
<group name="windows,firewall,portscan,">

  <!-- Fires on every single TCP DROP event from the Windows firewall log -->
  <rule id="100002" level="6">
    <if_sid>18145</if_sid>
    <match>DROP TCP</match>
    <description>Windows Firewall: TCP packet dropped</description>
    <group>firewall,</group>
  </rule>

  <!-- Fires when the same source IP triggers 10+ drops in 5 seconds = port scan -->
  <rule id="100003" level="10" frequency="10" timeframe="5">
    <if_matched_sid>100002</if_matched_sid>
    <same_source_ip />
    <description>Windows Firewall: Possible port scan detected - multiple dropped TCP packets from same source</description>
    <mitre>
      <id>T1046</id>
    </mitre>
    <group>portscan,pci_dss_11.4,</group>
  </rule>

</group>
```

Restart the Wazuh Manager to apply:

```bash
sudo systemctl restart wazuh-manager
```

📸 `[SCREENSHOT — local_rules.xml showing the new rule block]`

> **How the rule works:** Rule `100002` catches each individual DROP event. Rule `100003` acts as a frequency counter — if the same source IP triggers `100002` ten or more times within 5 seconds, Wazuh escalates to a level-10 alert tagged with MITRE T1046. This is the port scan detection trigger.

---

## 💥 Step 4 — Launch the Port Scan

With detection configured, we launch the attack from Kali.

**On Kali:**

```bash
nmap -sS 192.168.11.116
```

| Flag | What it does |
|------|-------------|
| `-sS` | SYN scan (stealth scan) — sends SYN packets without completing the TCP handshake |
| `192.168.11.116` | Target IP (Windows machine) |

📸 `[SCREENSHOT — Nmap output showing open ports discovered on the target]`

The scan completes and returns a list of open ports. On the Windows side, every probe that hit a closed or filtered port generated a firewall DROP entry — exactly what our rules are watching for.

---

## 📊 Step 5 — Observe the Attack in Splunk

### Dashboard before the attack

📸 `[SCREENSHOT — Splunk dashboard before the scan, baseline alert count, flat timeline]`

The dashboard shows a quiet baseline — normal background noise with no unusual spikes.

### Dashboard after the attack

📸 `[SCREENSHOT — Splunk dashboard after the scan, visible spike in the alert timeline]`

After the scan, a sharp spike appears on the timeline. This is the visual signature of a port scan: a sudden burst of DROP events from a single source in a very short window.

---

## 🔎 Step 6 — Investigate the Alert in Splunk

In **Splunk Search and Reporting**, run:

```
index="wazuh-alerts" agent.name="DESKTOP-OGUH4L2" rule.id=100003
```

📸 `[SCREENSHOT — Splunk search results showing the port scan alert]`

📸 `[SCREENSHOT — Alert detail view showing source IP, timestamp, and rule description]`

### What the alert tells us

| Field | Value |
|-------|-------|
| Rule ID | 100003 |
| Rule Level | 10 (High) |
| Description | Possible port scan detected — multiple dropped TCP packets from same source |
| Source IP | Kali Linux attacker IP |
| MITRE Tag | T1046 — Network Service Discovery |

> **The key signal:** A single source IP generating 10+ firewall DROP events in under 5 seconds is not normal user behavior. It is a machine cycling through ports systematically. That pattern is a port scan. An L1 analyst seeing this should immediately flag the source IP and escalate.

---

## 🧩 MITRE ATT&CK Mapping

| Field | Value |
|-------|-------|
| Tactic | Reconnaissance |
| Technique | T1046 — Network Service Discovery |
| Tool Used | Nmap (SYN scan) |
| Detection Method | Wazuh firewall drop correlation → Splunk alert spike |
| Rule Trigger | 10 TCP DROPs from same source IP within 5 seconds |

---

## 📋 Key Takeaways

- **Wazuh doesn't detect port scans out of the box** — it requires firewall logging to be enabled on the Windows agent and a custom frequency-based rule on the manager side. Understanding how to configure your own detection is a core SOC skill.
- **The visual signature matters** — a sharp spike in a flat Splunk timeline from a single source IP is one of the clearest triage signals an analyst will see. Recognizing patterns before diving into raw logs is what separates fast analysts from slow ones.
- **Catching the scan = stopping the kill chain** — a port scan means the attacker is still in the reconnaissance phase. Blocking the source IP at this stage prevents everything that comes after: brute force, exploitation, and persistence.

---

*by [ryo](https://github.com/RYO-1313)*
