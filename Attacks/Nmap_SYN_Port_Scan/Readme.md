## 📖 Introduction

### What is an Nmap SYN Scan?

An **Nmap SYN Scan** (also known as a **Half-Open Scan** or **Stealth Scan**) is one of the most popular and widely used port scanning techniques in network reconnaissance. When an attacker runs `nmap -sS`, the tool sends TCP SYN packets to the target machine on various ports:

- If the port is **open**, the target responds with a **SYN-ACK** packet.
- The attacker then immediately sends a **RST (Reset)** packet instead of completing the three-way handshake.
- If the port is **closed**, the target responds with a **RST** packet.

Because the TCP handshake is never completed, this scan is considered "stealthy" — it may evade older intrusion detection systems. However, modern SIEM solutions like Splunk can detect this activity through kernel-level firewall logs.

### What is Splunk Enterprise?

**Splunk Enterprise** is a powerful Security Information and Event Management (SIEM) platform that collects, indexes, and correlates machine-generated data from various sources in real time. It provides a web-based interface for searching, monitoring, and analyzing log data, making it an industry-standard tool for threat detection and incident response.

### Why is SIEM Monitoring Important?

In modern cybersecurity, attackers often perform reconnaissance silently before launching a full-scale attack. **SIEM monitoring is critical because:**

- It provides **real-time visibility** into network traffic and system events.
- It enables **early detection** of scanning, brute-force, and lateral movement activities.
- It creates an **audit trail** that supports forensic investigation.
- It allows security analysts to **correlate events** across multiple machines.

Without a SIEM, an Nmap SYN scan against your network could go completely unnoticed.

---

## 🏗️ Architecture Overview

### Lab Environment

| Role | Operating System | IP Address |
|------|-----------------|------------|
| 🔴 Attacker | Kali Linux | `192.168.56.103` |
| 🟡 Victim / Log Source | Ubuntu (with Splunk Universal Forwarder) | `192.168.56.104` |
| 🟢 SIEM Server | Windows (Splunk Enterprise) | `192.168.56.1` |

### Log Flow Diagram

```
┌─────────────────────┐        SYN Packets        ┌──────────────────────────┐
│   Kali Linux        │ ────────────────────────► │   Ubuntu (Victim)        │
│   (Attacker)        │                            │   192.168.56.104         │
│   192.168.56.103    │                            │                          │
└─────────────────────┘                            │  Kernel logs SYN packets │
                                                   │  into /var/log/syslog    │
                                                   │                          │
                                                   │  ┌────────────────────┐  │
                                                   │  │ Splunk Universal   │  │
                                                   │  │ Forwarder          │  │
                                                   │  │ Monitors syslog    │  │
                                                   │  └────────┬───────────┘  │
                                                   └───────────┼──────────────┘
                                                               │
                                                               │ TCP Port 9997
                                                               ▼
                                                   ┌──────────────────────────┐
                                                   │   Windows Host           │
                                                   │   Splunk Enterprise      │
                                                   │   192.168.56.1           │
                                                   │   Indexes & Analyzes     │
                                                   │   incoming log data      │
                                                   └──────────────────────────┘
```


## 🌐 Step 1 — Network Verification (Ping Test)

Before performing any scanning, it was essential to verify that **Kali Linux** and **Ubuntu** could communicate with each other on the same Host-Only network.

A ping test was performed from Kali Linux to Ubuntu:

```bash
ping 192.168.56.104
```

### Output

```
PING 192.168.56.104 (192.168.56.104) 56(84) bytes of data.
64 bytes from 192.168.56.104: icmp_seq=1 ttl=64 time=18.2 ms
64 bytes from 192.168.56.104: icmp_seq=2 ttl=64 time=1.79 ms
64 bytes from 192.168.56.104: icmp_seq=3 ttl=64 time=1.32 ms
64 bytes from 192.168.56.104: icmp_seq=4 ttl=64 time=3.94 ms
64 bytes from 192.168.56.104: icmp_seq=5 ttl=64 time=2.41 ms

--- 192.168.56.104 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4161ms
rtt min/avg/max/mdev = 1.322/5.536/18.219/6.402 ms
```

✅ **0% packet loss** confirmed that both machines were reachable within the VirtualBox Host-Only network.

> 📸 *Screenshot: Ping test from Kali Linux (192.168.56.103) to Ubuntu (192.168.56.104)*

---

## 🔎 Step 2 — Verifying IP Addresses

### Kali Linux — `ip a`

The IP address of the Kali Linux attacker machine was verified using:

```bash
ip a
```

The output confirmed that the **eth1** interface was assigned the IP address `192.168.56.103` on the Host-Only network.

> 📸 *Screenshot: `ip a` output on Kali Linux showing eth1 → 192.168.56.103*

---

### Ubuntu — `ifconfig`

On the Ubuntu victim machine, the IP address was verified using:

```bash
ifconfig
```

The **enp0s8** interface showed the IP address `192.168.56.104`, confirming it was on the same Host-Only network.

> 📸 *Screenshot: `ifconfig` output on Ubuntu showing enp0s8 → 192.168.56.104*

---

### Windows Splunk Server — `ipconfig`

On the Windows host running Splunk Enterprise, the IP address was verified using:

```cmd
ipconfig
```

The **VirtualBox Host-Only Network** adapter showed the IP address `192.168.56.1`, which serves as the Splunk Enterprise server address.

> 📸 *Screenshot: `ipconfig` output on Windows showing VirtualBox Host-Only adapter → 192.168.56.1*

---

## ⚙️ Step 3 — Splunk Universal Forwarder Setup

The **Splunk Universal Forwarder** was pre-installed on Ubuntu at `/opt/splunkforwarder/`. The forward server pointing to the Splunk Enterprise instance on Windows (`192.168.56.1:9997`) had already been configured.

To verify the current forwarding status, the following command was run from the Splunk forwarder binary directory:

```bash
cd /opt/splunkforwarder/bin
sudo ./splunk list forward-server
```

---

## ⚠️ Step 4 — Initial Problem: Inactive Forward Server

### Problem Encountered

When the forwarding status was checked, the output showed:

```
Active forwards:
        None
Configured but inactive forwards:
        192.168.56.1:9997
```

> 📸 *Screenshot: Terminal output showing `192.168.56.1:9997` configured but inactive*

### Root Cause

The forward server was **configured but inactive**, meaning the Splunk Universal Forwarder on Ubuntu was **unable to reach** the Splunk Enterprise server on Windows. After investigation, it was determined that **Windows Defender Firewall** was blocking all inbound TCP traffic on port **9997**, which is the default port used by Splunk forwarders to send data.

---

## 🛡️ Step 5 — Fixing the Issue: Windows Defender Firewall Configuration

To resolve the connectivity issue, a new **Inbound Rule** was created in **Windows Defender Firewall with Advanced Security** to allow TCP traffic on port 9997.

### Step-by-Step Firewall Rule Creation

**Step 5.1 — Open Windows Defender Firewall**

Navigate to:
`Control Panel → System and Security → Windows Defender Firewall`

The firewall was active on the Public network profile, blocking all inbound connections by default.

> 📸 *Screenshot: Windows Defender Firewall showing Public network — Incoming connections blocked*

---

**Step 5.2 — Open Advanced Security Settings**

Click on **Advanced settings** to open **Windows Defender Firewall with Advanced Security**. This confirms:
- Firewall is ON for Domain, Private, and Public profiles.
- Inbound connections that do not match a rule are **blocked**.

> 📸 *Screenshot: Windows Defender Firewall with Advanced Security overview*

---

**Step 5.3 — Navigate to Inbound Rules**

Click on **Inbound Rules** in the left panel, then click **New Rule...** in the Actions panel on the right.

> 📸 *Screenshot: Inbound Rules panel with New Rule option*

---

**Step 5.4 — Rule Type: Port**

In the New Inbound Rule Wizard, select **Port** as the rule type.

| Setting | Value |
|---------|-------|
| Rule Type | Port |

> 📸 *Screenshot: New Inbound Rule Wizard — Rule Type set to "Port"*

---

**Step 5.5 — Protocol and Ports**

Select **TCP** and specify **Specific local ports** as `9997`.

| Setting | Value |
|---------|-------|
| Protocol | TCP |
| Specific local ports | 9997 |

> 📸 *Screenshot: New Inbound Rule Wizard — TCP selected, port 9997 entered*

---

**Step 5.6 — Action: Allow the Connection**

Select **Allow the connection** to permit Splunk forwarder traffic on port 9997.

| Setting | Value |
|---------|-------|
| Action | Allow the connection |

> 📸 *Screenshot: New Inbound Rule Wizard — "Allow the connection" selected*

---

**Step 5.7 — Profile: Apply to All Profiles**

Leave all three profiles checked so the rule applies regardless of network type.

| Profile | Applied |
|---------|---------|
| Domain | ✅ Yes |
| Private | ✅ Yes |
| Public | ✅ Yes |

> 📸 *Screenshot: New Inbound Rule Wizard — All profiles (Domain, Private, Public) checked*

---

**Step 5.8 — Name the Rule**

Name the rule `Splunk9997` to clearly identify it.

| Setting | Value |
|---------|-------|
| Rule Name | Splunk9997 |

Click **Finish** to create the rule.

> 📸 *Screenshot: New Inbound Rule Wizard — Rule named "Splunk9997", ready to finish*

---

## 🔄 Step 6 — Restarting Services

### Restart Splunk Enterprise (Windows)

After creating the firewall rule, the **Splunk Enterprise** service was restarted from the Windows **Services** application (`services.msc`) to ensure the changes took effect and the server was ready to receive forwarded logs.

### Restart Splunk Universal Forwarder (Ubuntu)

The Splunk forwarder was restarted on Ubuntu using:

```bash
cd /opt/splunkforwarder/bin
sudo ./splunk restart
```

### Restart Output

```
Warning: Attempting to revert the SPLUNK_HOME ownership
Warning: Executing "chown -R splunkfwd:splunkfwd /opt/splunkforwarder"
Stopping splunkd...
Shutting down.  Please wait, as this may take a few minutes.
Stopping splunk helpers...
Done.

Splunk> The Notorious B.I.G. D.A.T.A.

Checking prerequisites...
        Checking mgmt port [8089]: open
New certs have been generated in '/opt/splunkforwarder/etc/auth'.
        Checking conf files for problems...
        Done
        Checking default conf files for edits...
        Validating installed files...
        All installed files intact.
All preliminary checks passed.

Starting splunk server daemon (splunkd)...
Done
```

> 📸 *Screenshot: Ubuntu terminal showing successful Splunk forwarder restart*

---

## ✅ Step 7 — Successful Connection Verification

After restarting both services, the forward server status was rechecked:

```bash
sudo ./splunk list forward-server
```

### Output (After Fix)

```
Active forwards:
        192.168.56.1:9997
Configured but inactive forwards:
        None
```

> 📸 *Screenshot: Terminal showing `192.168.56.1:9997` now listed under Active forwards*

🎉 The forward server status changed from **inactive → active**, confirming that the **Ubuntu Splunk Universal Forwarder** was successfully communicating with the **Windows Splunk Enterprise** server on port 9997.

**The root cause** (Windows Defender Firewall blocking port 9997) was fully resolved by adding the `Splunk9997` inbound rule.

---

## 💻 Step 8 — Performing the Nmap SYN Scan

With the logging infrastructure confirmed to be working, the Nmap SYN scan was launched from **Kali Linux** against the **Ubuntu victim machine**.

### Command Used

```bash
sudo nmap -sS 192.168.56.104
```

### How a SYN Scan Works

```
Kali Linux                          Ubuntu (Victim)
    │                                     │
    │ ──── SYN (port 22) ──────────────► │
    │ ◄─── SYN-ACK ──────────────────── │  (Port OPEN)
    │ ──── RST ───────────────────────► │  (Connection aborted — never completed)
    │                                     │
    │ ──── SYN (port 80) ──────────────► │
    │ ◄─── RST ───────────────────────── │  (Port CLOSED)
    │                                     │
    │ ──── SYN (port 443) ─────────────► │
    │ ◄─── RST ───────────────────────── │  (Port CLOSED)
```

| Scan Characteristic | Description |
|---------------------|-------------|
| Scan Type | Half-Open / Stealth Scan |
| Packets Sent | TCP SYN |
| Open Port Response | SYN-ACK (then RST sent by attacker) |
| Closed Port Response | RST |
| Handshake Completed? | ❌ No |

> ⚠️ SYN scans require **root/sudo** privileges because they require raw socket access.

---

## 📊 Step 9 — Scan Results

### Nmap Output

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-06 11:12 IST
Nmap scan report for 192.168.56.104
Host is up (0.032s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE   SERVICE
22/tcp  open    ssh
80/tcp  closed  http
443/tcp closed  https
MAC Address: 08:00:27:98:91:15 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 21.96 seconds
```

> 📸 *Screenshot: Nmap SYN scan results from Kali Linux terminal*

### Analysis of Results

| Port | State | Service | Interpretation |
|------|-------|---------|----------------|
| 22/tcp | Open | SSH | SSH service is actively listening — received SYN-ACK |
| 80/tcp | Closed | HTTP | HTTP port not in use — received RST |
| 443/tcp | Closed | HTTPS | HTTPS port not in use — received RST |

The 997 filtered ports returned no response, indicating they are behind a firewall.

---

## 📦 Step 10 — Log Collection Process

When the Nmap SYN scan was executed, the following log collection pipeline was triggered:

```
1. Kali Linux sent SYN packets to 192.168.56.104
         ↓
2. Ubuntu's kernel (via firewall rules) logged each
   SYN packet as a [UFW BLOCK] entry in /var/log/syslog
         ↓
3. Splunk Universal Forwarder (running on Ubuntu)
   continuously monitored /var/log/syslog for new entries
         ↓
4. New log lines were forwarded in real time over TCP port 9997
   to Splunk Enterprise on Windows (192.168.56.1)
         ↓
5. Splunk Enterprise indexed the forwarded logs
   and made them searchable for analysis
```

Each SYN packet to a different destination port generated a **separate log entry**, creating a clear pattern of port scanning activity in the SIEM.

---

## 🔍 Step 11 — Searching Logs in Splunk Enterprise

### Search by Attacker IP

In the Splunk Search & Reporting app, the following search was used to find all events originating from the Kali Linux attacker:

```spl
192.168.56.103
```

This returned **20 events** within the last 24 hours, all timestamped around the time the scan was performed (`2026-05-06T11:12:xx`).

### Search by Source Log File

```spl
source="/var/log/syslog"
```

This search confirmed all logs were sourced from the Ubuntu machine's syslog file, forwarded by the Universal Forwarder.

> 📸 *Screenshot: Splunk Enterprise Search showing 20 events from 192.168.56.103 with UFW BLOCK entries in /var/log/syslog*

---

## 🧪 Step 12 — Log Analysis

### Sample Log Entry

```
2026-05-06T11:12:15.819062+05:30 vamsi-VirtualBox kernel: 
[UFW BLOCK] IN=enp0s8 OUT= MAC=08:00:27:98:91:15:08:00:27:01:11:38:08:00 
SRC=192.168.56.103 DST=192.168.56.104 LEN=44 TOS=0x00 PREC=0x00 
TTL=48 ID=45826 PROTO=TCP SPT=41429 DPT=199 WINDOW=1024 RES=0x00 SYN URGP=0
```

### Field Breakdown

| Field | Value (Example) | Meaning |
|-------|----------------|---------|
| `[UFW BLOCK]` | — | Packet was blocked by the firewall rule |
| `IN` | `enp0s8` | Network interface that received the packet |
| `SRC` | `192.168.56.103` | **Source IP — Kali Linux (Attacker)** |
| `DST` | `192.168.56.104` | **Destination IP — Ubuntu (Victim)** |
| `PROTO` | `TCP` | Protocol used — TCP confirms SYN scan |
| `SPT` | `41429` | Source Port (randomly assigned by Nmap) |
| `DPT` | `199` | **Destination Port — the port being scanned** |
| `SYN` | present | **SYN flag set — confirms this is a SYN packet** |
| `WINDOW` | `1024` | TCP window size (fixed value typical of Nmap) |
| `URGP` | `0` | No urgent pointer set |
| `TTL` | `48` | Time-to-live of the packet |

### Why This Indicates Port Scanning

The critical indicator in these logs is the **`DPT` (Destination Port) field**. Across the 20 events captured, the `DPT` value was **different in every single log entry** — ranging across ports like 199, 111, 587, 993, 143, 1723, and many others. This pattern of:

- **Same source IP** (`192.168.56.103`)
- **Same destination IP** (`192.168.56.104`)
- **Same time window** (within milliseconds of each other)
- **Different destination ports** in rapid succession
- **SYN flag** present in every packet

...is the textbook signature of an **Nmap SYN port scan**. The Splunk SIEM successfully captured and displayed all of this evidence in a single searchable interface.

---

## 🏁 Conclusion

This lab successfully demonstrated the complete lifecycle of an **Nmap SYN Port Scan attack** — from execution to detection using **Splunk Enterprise** as a SIEM platform.

### Key Achievements

| Achievement | Status |
|-------------|--------|
| Network connectivity verified between all VMs | ✅ |
| Splunk Universal Forwarder installed on Ubuntu | ✅ |
| Windows Firewall port 9997 opened for log forwarding | ✅ |
| Forwarder successfully connected to Splunk Enterprise | ✅ |
| Nmap SYN scan executed from Kali Linux | ✅ |
| Scan logs captured in Ubuntu syslog | ✅ |
| Logs forwarded to Splunk Enterprise in real time | ✅ |
| SYN scan pattern identified in Splunk search | ✅ |

### Key Lessons Learned

**1. Firewall Configuration is Critical for SIEM Functionality**
The initial failure of log forwarding was caused by **Windows Defender Firewall blocking port 9997**. This highlights that even the defensive infrastructure itself must be properly configured — an incorrectly configured SIEM is as bad as no SIEM at all.

**2. SYN Scans Leave Clear Evidence in Kernel Logs**
Despite being called a "stealth scan," the Nmap SYN scan generated distinct `[UFW BLOCK]` entries in `/var/log/syslog` for every port probed. The `SYN` flag and rapidly changing `DPT` values across entries from the same source IP are unmistakable indicators of reconnaissance.

**3. SIEM Monitoring Provides Actionable Intelligence**
Splunk's ability to search, filter, and correlate 20 log events in real time demonstrates how a SIEM transforms raw log noise into actionable security intelligence. Without Splunk, this attack may have been buried in system logs and gone unnoticed.

**4. Defense-in-Depth Works**
This lab simulates a complete security monitoring pipeline — a network with multiple layers (host-based firewall, log forwarding, SIEM analysis) where each layer contributes to overall visibility and protection.

---

## 🛠️ Tools & Technologies Used

| Tool | Version / Details | Purpose |
|------|-------------------|---------|
| Oracle VirtualBox | Host-Only Network | Lab environment isolation |
| Kali Linux | Rolling | Attacker machine |
| Ubuntu | 22.04 LTS | Victim / log source machine |
| Windows | Windows 10/11 | Splunk Enterprise host |
| Nmap | 7.95 | SYN port scanning |
| Splunk Universal Forwarder | 10.2.2 | Log shipping from Ubuntu |
| Splunk Enterprise | Latest | SIEM — log indexing and analysis |
| Windows Defender Firewall | Built-in | Inbound rule configuration |

---
