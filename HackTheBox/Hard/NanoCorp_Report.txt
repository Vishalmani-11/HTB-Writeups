================================================================================
                    NANOCORP.HTB - FULL PENETRATION TEST REPORT
================================================================================

Target IP          : 10.129.243.199 (also seen at 10.129.19.168, 10.129.19.172)
Hostname           : dc01.nanocorp.htb
OS                 : Windows Server 2022 Standard (Build 20348) x64
Domain             : nanocorp.htb
Domain SID         : S-1-5-21-2261381271-1331810270-697239744
Difficulty         : Hard
Environment        : Active Directory (Single Domain Controller)
Engagement Type    : Offensive Security Assessment (HTB CTF)
Author             : vishalgodseye
Date               : 2026-06-08 to 2026-06-09

Box Changelog (as of November 2025):
  - CVE-2025-54918 patched
  - CVE-2025-33073 patched
  Note: These patches may have altered the original intended attack path.

================================================================================
                        1. EXECUTIVE SUMMARY
================================================================================

This assessment targeted the NanoCorp Active Directory environment hosted
on a Windows Server 2022 Domain Controller. The engagement began with a
black-box network penetration test against a single target host.

The attack chain started with SMB null session authentication, which revealed
anonymous access was permitted. Web application enumeration discovered a file
upload feature on the hiring portal (Apache/PHP) that accepted ZIP archives.
By leveraging CVE-2025-24071 (Windows SMB NTLM hash capture via malicious ZIP
structure), we captured NTLMv2-SSP hashes of the web_svc service account.
Cracking these hashes revealed valid domain credentials. BloodHound analysis
identified an ACL-based privilege escalation path: web_svc had AddSelf rights
to the IT_SUPPORT group, which in turn held ForceChangePassword permission on
the monitoring_svc account. We successfully abused the AddSelf membership,
reset the monitoring_svc password, obtained a Kerberos TGT, and gained
remote shell access to the domain controller via Evil-WinRM. The user flag
was captured from the monitoring_svc desktop.

Privilege escalation to SYSTEM/Administrator was attempted via CVE-2024-0670
(CheckMK Agent MSI repair local privilege escalation) but was not completed
during this engagement due to file permission issues on the target.

+----------+--------+--------------------------------------------------+-----------+
| Severity | Rating | Finding                                          | Status    |
+----------+--------+--------------------------------------------------+-----------+
| HIGH     | 8.5    | CVE-2025-24071 NTLM hash capture via ZIP upload  | Exploited |
| MEDIUM   | 6.5    | SMB Null Session Authentication                  | Confirmed |
| MEDIUM   | 6.0    | Weak Cracked Password (web_svc)                  | Exploited |
| HIGH     | 8.0    | ACL Abuse: AddSelf -> IT_SUPPORT                 | Exploited |
| HIGH     | 8.5    | ACL Abuse: ForceChangePassword on monitoring_svc | Exploited |
| HIGH     | 8.8    | CVE-2024-0670 CheckMK MSI Repair LPE (potential) | Identified|

================================================================================
                        2. SCOPE & METHODOLOGY
================================================================================

+---------------+-------------------------------------------------------------+
| Target        | 10.129.243.199 (dc01.nanocorp.htb)                        |
| Open Ports    | 53, 80, 88, 135, 139, 389, 593, 636, 3268, 3269, 5986,    |
|               | 9389, 49664, 49668, 50364, 50369, 50393, 55227             |
| Web Apps      | Hiring Portal (Apache 2.4.58, PHP 8.2.12) on port 80       |
| Exclusions    | Denial of Service, Social Engineering                      |
+---------------+-------------------------------------------------------------+

Methodology Phases:
  1. Network Discovery & Port Scanning
  2. Service Enumeration (SMB, LDAP, HTTP)
  3. Vulnerability Identification
  4. Initial Access (CVE-2025-24071 exploitation)
  5. Post-Exploitation & Credential Access
  6. Active Directory Enumeration
  7. Privilege Escalation via ACL Abuse
  8. BloodHound Attack Path Analysis
  9. Shell Access via Kerberos Authentication
  10. Local Privilege Escalation Attempt (CVE-2024-0670)

Tools Used:
  - RustScan 2.x (port scanning)
  - Nmap 7.99 (service/version detection)
  - NetExec (nxc) 1.x (SMB/LDAP enumeration)
  - Responder 3.2.0 (NTLM hash capture)
  - John the Ripper (hash cracking)
  - Impacket getTGT.py (Kerberos TGT acquisition)
  - BloodHound-CE-Python (AD attack graph)
  - BloodyAD (ACL abuse)
  - smbclient (SMB share enumeration)
  - ldapsearch (LDAP enumeration)
  - Evil-WinRM 3.9 (remote shell via WinRM)
  - CVE-2025-24071 PoC by 0x6rss (malicious ZIP generation)
  - CVE-2024-0670 PoC by tralsesec (CheckMK MSI repair exploit)
  - RunasCs by antonioCoco (user context switching)

================================================================================
                        3. ENUMERATION
================================================================================

3.1 Port Scan Results (Nmap Aggressive Scan)
---------------------------------------------

+-------+----------+------------------+--------------------------------------+
| Port  | Protocol | Service          | Version / Notes                      |
+-------+----------+------------------+--------------------------------------+
| 53    | tcp      | domain           | Simple DNS Plus                      |
| 80    | tcp      | http             | Apache httpd 2.4.58 (OpenSSL/3.1.3  |
|       |          |                  | PHP/8.2.12) Win64                    |
| 88    | tcp      | kerberos-sec     | Microsoft Windows Kerberos           |
| 135   | tcp      | msrpc            | Microsoft Windows RPC                |
| 139   | tcp      | netbios-ssn      | Microsoft Windows netbios-ssn        |
| 389   | tcp      | ldap             | AD LDAP (Domain: nanocorp.htb)       |
| 593   | tcp      | ncacn_http       | Microsoft Windows RPC over HTTP 1.0  |
| 636   | tcp      | ldapssl          | LDAP over SSL                        |
| 3268  | tcp      | ldap             | AD LDAP Global Catalog               |
| 3269  | tcp      | globalcatLDAPssl | Global Catalog SSL                   |
| 5986  | tcp      | wsmans           | WinRM/WS-Management (SSL)            |
| 9389  | tcp      | adws             | AD Web Services                      |
| 49664 | tcp      | unknown          | RPC dynamic port                     |
| 49668 | tcp      | unknown          | RPC dynamic port                     |
| 50364 | tcp      | ncacn_http       | Microsoft Windows RPC over HTTP 1.0  |
| 50369 | tcp      | unknown          | RPC dynamic port                     |
| 50393 | tcp      | unknown          | RPC dynamic port                     |
| 55227 | tcp      | unknown          | RPC dynamic port                     |
+-------+----------+------------------+--------------------------------------+

Service Info: Host: DC01, OS: Windows, CPE: cpe:/o:microsoft:windows

SSL Certificate (port 5986):
  Subject: CN=dc01.nanocorp.htb
  SAN: DNS:dc01.nanocorp.htb
  Issuer: CN=dc01.nanocorp.htb (self-signed)
  Valid: 2025-04-06 to 2026-04-06 (EXPIRED)
  Public Key: RSA 2048-bit
  Signature: sha256WithRSAEncryption
  MD5:    2e3e1a1010b87f43dc93a4d905ef6053
  SHA-1:  4674631227cee78391b7ec001746f114d6694ea0

Nmap Host Scripts:
  - smb2-security-mode: Could not find NetBIOS name for server
  - p2p-conficker: 0/4 positive (CLEAN)
  - smb2-time: Script execution failed


3.2 SMB Enumeration
-------------------

NetExec Null Authentication Check:
  Server: 10.129.243.199:445 (DC01)
  OS: Windows Server 2022 Build 20348 x64
  Domain: nanocorp.htb
  Signing: True
  SMBv1: None
  Null Auth: TRUE

  Finding: Anonymous NULL session login is permitted on SMB.

SMB Share enumeration with null auth failed:
  Error: STATUS_USER_SESSION_DELETED

SMB Share enumeration with web_svc credentials:
  Sharename       Type    Comment
  ----------      ----    -------
  ADMIN$          Disk    Remote Admin
  C$              Disk    Default share
  IPC$            IPC     Remote IPC
  NETLOGON        Disk    Logon server share
  SYSVOL          Disk    Logon server share


3.3 LDAP Enumeration
--------------------

RootDSE (anonymous bind, scope base):
  domainFunctionality: 7
  forestFunctionality: 7
  domainControllerFunctionality: 7
  rootDomainNamingContext: DC=nanocorp,DC=htb
  ldapServiceName: nanocorp.htb:dc01$@NANOCORP.HTB
  dnsHostName: DC01.nanocorp.htb
  defaultNamingContext: DC=nanocorp,DC=htb

Domain Users (LDAP authenticated with web_svc, 5 total):
  Username          | Last PW Set          | BadPW | Description
  -----------------|----------------------|-------|---------------------------
  Administrator     | 2025-04-10 04:30:49  | 0     | Built-in admin account
  Guest             | <never>              | 0     | Built-in guest account
  krbtgt            | 2025-04-03 07:08:45  | 0     | KDC service account
  web_svc           | 2025-04-10 04:29:38  | 0     | Service account
  monitoring_svc    | 2026-06-09 02:38:55  | 0     | Monitoring service account

Domain Groups (LDAP authenticated, key groups):
  - Administrators (3 members)
  - IIS_IUSRS (1 member)
  - Remote Management Users (1 member)
  - Protected Users (1 member)
  - IT_Support (0 members initially, 1 after AddSelf abuse)
  - Schema Admins (1 member)
  - Enterprise Admins (1 member)
  - Domain Admins (1 member: Administrator)
  - Group Policy Creator Owners (1 member)
  - Pre-Windows 2000 Compatible Access (1 member)
  - Windows Authorization Access Group (1 member)
  - Denied RODC Password Replication Group (8 members)

Domain Computers: DC01$ (1 computer)


3.4 Web Application Enumeration
---------------------------------

Port 80 - Hiring Portal:
  Platform: Apache 2.4.58 (Win64), OpenSSL 3.1.3, PHP 8.2.12
  Subdomain found: hiring.nanocorp.htb
  Functionality: Accepts file uploads (ZIP format) for candidate applications
  Vulnerability: ZIP upload accepts crafted archives for NTLM hash capture


3.5 Kerberoasting
------------------

Kerberoast attempt against nanocorp.htb:
  Total records returned: 0
  Finding: No accounts with crackable SPNs configured.


3.6 BloodHound Analysis
------------------------

BloodHound data collected via SharpHound, analyzed with BloodHound-CE.

Key ACL Findings from BloodHound JSON data:

IT_SUPPORT Group (SID: S-1-5-21-...-3102):
  ACEs:
  - Domain Admins (SID -512): Owns, GenericAll
  - web_svc (SID -1103): AddSelf (non-inherited)          <-- CRITICAL
  - Account Operators (S-1-5-32-548): GenericAll
  - Enterprise Admins (SID -519): GenericAll (inherited)
  - Administrators (S-1-5-32-544): GenericWrite, WriteOwner, WriteDacl (inherited)
  Members: (empty before exploitation)

monitoring_svc User (SID: S-1-5-21-...-3101):
  Member Of:
  - Protected Users (SID -525): Additional authentication protections
  - Remote Management Users (SID -580): WinRM access enabled
  - Domain Users (SID -513): Default domain group
  ACEs:
  - Domain Admins (SID -512): Owns, GenericAll
  - IT_SUPPORT (SID -3102): ForceChangePassword (inherited) <-- CRITICAL
  - Key Admins (SID -526): AddKeyCredentialLink (inherited)
  - Enterprise Key Admins (SID -527): AddKeyCredentialLink (inherited)
  - Enterprise Admins (SID -519): GenericAll (inherited)
  - Administrators (S-1-5-32-544): GenericWrite, WriteOwner, AllExtendedRights, WriteDacl
  - Account Operators (S-1-5-32-548): GenericAll
  Properties:
  - pwdneverexpires: True
  - dontreqpreauth: False
  - hasspn: False
  - enabled: True
  - admincount: False

web_svc User (SID: S-1-5-21-...-1103):
  ObjectIdentifier: S-1-5-21-2261381271-1331810270-697239744-1103
  Description: (none)

Identified Attack Path:
  WEB_SVC --[AddSelf]--> IT_SUPPORT --[ForceChangePassword]--> MONITORING_SVC


================================================================================
                        4. ATTACK CHAIN & VULNERABILITIES
================================================================================

VULNERABILITY 1: CVE-2025-24071 - Malicious ZIP NTLM Hash Disclosure
----------------------------------------------------------------------
CVE ID       : CVE-2025-24071
CVSS Score   : 7.5 (High)
Location     : Hiring portal ZIP upload (http://hiring.nanocorp.htb, port 80)
Description  : The hiring portal accepted ZIP file uploads. A specially
               crafted ZIP archive containing a Windows Search shortcut (.lnk)
               that references a remote SMB share triggers the server-side
               application to authenticate to an attacker-controlled SMB server,
               leaking the NTLMv2-SSP hash of the processing service account.
Impact       :
  - Unauthenticated remote attacker can capture NTLM hashes
  - Hashes can be cracked offline to obtain plaintext credentials
  - Leads to initial domain access
Remediation  :
  - Patch Windows Server per Microsoft advisory for CVE-2025-24071
  - Disable SMB client-side automatic outbound connections
  - Restrict ZIP upload processing to sandboxed environments only
  - Implement NTLM relay protections (EPA, SMB signing enforcement)

Exploitation Steps:
  1. git clone https://github.com/0x6rss/CVE-2025-24071_PoC
  2. Generate malicious ZIP with embedded .lnk pointing to attacker IP
  3. Start Responder: sudo responder -I tun0 -v
  4. Upload crafted ZIP to the hiring portal
  5. Responder captured multiple NTLMv2-SSP hashes for NANOCORP\web_svc

Captured Hash:
  web_svc::NANOCORP:a6d5cf6beea6db51:3EB2B07424B812ACF93D17A5448FF18B:...

Cracked Credentials:
  Username: web_svc
  Password: dksehdgh712!@#
  Tool: John the Ripper with rockyou.txt wordlist
  Time: < 1 second


VULNERABILITY 2: SMB Null Session Authentication
-------------------------------------------------
CVE ID       : N/A
CVSS Score   : 5.3 (Medium)
Location     : SMB service (10.129.243.199:445)
Description  : The SMB service on the domain controller permitted anonymous
               null session authentication. While share enumeration was
               partially restricted (STATUS_USER_SESSION_DELETED), the null
               auth still confirmed domain structure and OS details.
Impact       :
  - Unauthenticated enumeration of domain names and OS information
  - Facilitates targeted attacks against the domain
Remediation  :
  - Disable null session authentication via Group Policy
  - Configure "Network access: Do not allow anonymous enumeration" policies
  - Restrict SMB access to authenticated domain accounts only


VULNERABILITY 3: Weak Password Policy and Cracked Service Account Credentials
------------------------------------------------------------------------------
CVE ID       : N/A
CVSS Score   : 6.5 (Medium)
Location     : Domain account web_svc
Description  : The web_svc service account password was weak enough to be
               cracked from captured NTLMv2-SSP hashes using a standard
               rockyou wordlist in under one second. This provides direct
               domain authentication.
               Cracked credentials: web_svc / dksehdgh712!@#
Impact       :
  - Unauthorized domain access with valid service account credentials
  - Credential can be used for Kerberoasting, LDAP queries, and lateral
    movement
Remediation  :
  - Enforce strong password policies for service accounts (16+ chars)
  - Implement gMSA (Group Managed Service Accounts) to auto-rotate passwords
  - Monitor for credential dumping and unauthorized hash capture


VULNERABILITY 4: Excessive ACL Permissions - AddSelf on IT_SUPPORT
-------------------------------------------------------------------
CVE ID       : N/A
CVSS Score   : 8.0 (High)
Location     : Active Directory group IT_SUPPORT
Description  : The web_svc account possessed the AddSelf permission on the
               IT_SUPPORT security group. This allowed the account to add
               itself to the group, inheriting all permissions granted to
               IT_SUPPORT.
Impact       :
  - Privilege escalation through group membership manipulation
  - Inherited ForceChangePassword on monitoring_svc account
  - Potential for further abuse of IT_SUPPORT group rights
Remediation  :
  - Audit all ACLs on privileged and delegated groups
  - Remove AddSelf permissions from service accounts
  - Implement privileged access management (PAM) solutions
  - Regular review of group memberships via scheduled audits

Exploitation Steps:
  1. Analyzed BloodHound attack graph:
     WEB_SVC -> AddSelf -> IT_SUPPORT -> ForceChangePassword -> MONITORING_SVC
  2. Acquired TGT: getTGT.py nanocorp.htb/web_svc:dksehdgh712!@# -dc-ip TARGET
  3. Fixed Kerberos clock skew: sudo timedatectl set-ntp false && ntpdate TARGET
  4. Added self to group: bloodyAD add groupMember it_support web_svc
  5. Verified membership via LDAP query on IT_Support group member attribute
  6. Result: Successfully escalated privileges via ACL abuse


VULNERABILITY 5: ForceChangePassword ACL on monitoring_svc
-----------------------------------------------------------
CVE ID       : N/A
CVSS Score   : 8.5 (High)
Location     : Active Directory user monitoring_svc
Description  : The IT_SUPPORT group possessed the ForceChangePassword extended
               right on the monitoring_svc account. After adding web_svc to
               IT_SUPPORT, we successfully reset the monitoring_svc password
               and obtained a Kerberos TGT for the account, gaining remote
               shell access to the domain controller.
Impact       :
  - Unauthorized password reset of a service account
  - monitoring_svc belongs to "Protected Users" and "Remote Management Users"
  - Full remote shell access to the domain controller via WinRM (port 5986)
  - User flag captured from monitoring_svc desktop
Remediation  :
  - Remove ForceChangePassword ACL from IT_SUPPORT on monitoring_svc
  - Move monitoring_svc to a group with stricter ACL hygiene
  - Implement tiered administration model preventing cross-tier password resets
  - Monitor for unauthorized password reset attempts in Security Event Log

Exploitation Steps:
  1. Reset password via bloodyAD:
     bloodyAD --host dc01.nanocorp.htb -d nanocorp.htb -u web_svc \
       -p 'dksehdgh712!@#' -k set password monitoring_svc 'P@ssw0rd444!'
  2. Result: [+] Password changed successfully!
  3. Acquired TGT for monitoring_svc:
     getTGT.py -dc-ip TARGET 'nanocorp.htb/monitoring_svc:P@ssw0rd444!'
  4. Exported credential cache: export KRB5CCNAME=$(pwd)/monitoring_svc.ccache
  5. Verified TGT: klist (valid ticket confirmed)
  6. Gained shell via Evil-WinRM (port 5986, SSL, Kerberos):
     evil-winrm -S -i dc01.nanocorp.htb -r NANOCORP.HTB
     Note: Port 5985 (HTTP) was filtered; only 5986 (HTTPS) was accessible
  7. Shell established: Evil-WinRM PS C:\Users\monitoring_svc\Documents>
  8. User flag captured from monitoring_svc desktop


VULNERABILITY 6: CVE-2024-0670 - CheckMK Agent MSI Repair LPE
-----------------------------------------------------------------
CVE ID       : CVE-2024-0670
CVSS Score   : 8.8 (High)
Location     : CheckMK Windows Agent (installed on dc01.nanocorp.htb)
Description  : The CheckMK Windows Agent MSI installer uses predictable
               filenames in C:\Windows\Temp during repair operations. An
               unprivileged user can pre-seed malicious .cmd files with
               predictable names. When the MSI repair runs (triggered via
               msiexec /fa), the installer executes these files as
               NT AUTHORITY\SYSTEM, granting local privilege escalation.
Impact       :
  - Local unprivileged user can escalate to NT AUTHORITY\SYSTEM
  - Full system compromise from any local user context
Remediation  :
  - Update CheckMK Agent to patched version
  - Restrict write access to C:\Windows\Temp
  - Monitor for suspicious .cmd file creation in temp directories

CheckMK Discovery Method:
  After obtaining the monitoring_svc shell, local enumeration was performed
  using WinPEAS and manual PowerShell commands. A non-Microsoft service
  "CheckMK Agent" was identified as unusual. Further investigation via
  registry enumeration confirmed the MSI installation:
    Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\CurrentVersion\Installer\...'
  Research revealed CVE-2024-0670, a local privilege escalation in the
  CheckMK Agent MSI repair mechanism.

Exploitation Attempt:
  1. Identified CheckMK installed via registry enumeration
  2. Downloaded exploit.ps1 (tralsesec/CVE-2024-0670), nc.exe, RunasCs.exe
  3. Used RunasCs to execute exploit.ps1 as web_svc (needed for C:\Windows\Temp write)
  4. Issue: monitoring_svc could not write to C:\Windows\Temp
  5. Workaround attempted: Used C:\ProgramData as alternative path
  6. Status: Exploit triggered but SYSTEM shell not captured
  7. Root cause: Seed files MUST be in C:\Windows\Temp for MSI repair to find them;
     web_svc context via RunasCs should have write access but path was changed
     to C:\ProgramData which the MSI repair process does not check


VULNERABILITY 7: Insufficient Kerberos Clock Skew Handling
-----------------------------------------------------------
CVE ID       : N/A
CVSS Score   : 3.5 (Low)
Location     : Kerberos authentication (port 88)
Description  : The client machine clock was approximately 7 hours out of sync
               with the domain controller, causing KRB_AP_ERR_SKEW errors.
               While not a direct vulnerability on the target, time
               synchronization is critical for Kerberos and should be properly
               configured on all domain-joined systems.
Impact       :
  - Authentication failures due to clock skew
  - Potential for replay attacks if clock skew tolerance is too permissive
Remediation  :
  - Configure NTP synchronization across all domain systems
  - Set max clock skew tolerance to 5 minutes (default) and monitor deviations
  - Implement chrony or Windows Time Service with proper domain hierarchy

================================================================================
                        5. FLAGS CAPTURED
================================================================================

+--------------------------------------------------------------------------+
| USER FLAG                                                                |
+--------------------------------------------------------------------------+
| Location : C:\Users\monitoring_svc\Desktop\user.txt                      |
| Method   : Evil-WinRM shell as monitoring_svc via Kerberos auth          |
| Source   : Password reset via ACL abuse (ForceChangePassword)            |
+--------------------------------------------------------------------------+

+--------------------------------------------------------------------------+
| ROOT FLAG                                                                |
+--------------------------------------------------------------------------+
| Location : C:\Users\Administrator\Desktop\root.txt                       |
| Method   : Not captured during this engagement                           |
| Status   : Requires SYSTEM/Administrator privilege escalation             |
+--------------------------------------------------------------------------+

BloodHound Attack Path (Fully Exploited):
  WEB_SVC --[AddSelf]--> IT_SUPPORT --[ForceChangePassword]--> MONITORING_SVC

  Steps completed:
  [+] web_svc authenticated to nanocorp.htb
  [+] TGT acquired for web_svc
  [+] Added web_svc to IT_SUPPORT group (AddSelf abuse)
  [+] Verified IT_SUPPORT membership via LDAP
  [+] ForceChangePassword ACL confirmed on monitoring_svc
  [+] Reset monitoring_svc password to P@ssw0rd444!
  [+] Acquired TGT for monitoring_svc
  [+] Established Evil-WinRM shell on dc01.nanocorp.htb as monitoring_svc
  [+] Captured user flag from monitoring_svc desktop

================================================================================
                        6. REMAINING ATTACK SURFACE
================================================================================

The following potential attack vectors and escalation paths remain unexplored:

1. Domain Administrator Access: The monitoring_svc account has been compromised
   but does not currently hold Domain Admin rights. Further privilege
   escalation to Administrator may be possible through additional ACL abuse,
   token impersonation, or resource delegation attacks.

2. CVE-2024-0670 Completion: The CheckMK MSI repair exploit was attempted but
   not completed. The correct approach would be to ensure the exploit runs
   in a context that can write to C:\Windows\Temp (the web_svc account via
   RunasCs should work, but the exploit.ps1 seed path must remain as
   C:\Windows\Temp\cmk_all_*.cmd, not C:\ProgramData).

3. Lateral Movement: Only one domain computer (DC01) was identified. If
   additional hosts exist, lateral movement from the compromised DC could
   extend access across the network.

4. Additional ACL Abuse on IT_SUPPORT: Beyond ForceChangePassword on
   monitoring_svc, the IT_SUPPORT group may have additional ACLs on other
   objects (WriteDACL, WriteOwner, GenericWrite, GenericAll) that were not
   fully enumerated.

5. SMB Share Contents: With web_svc credentials, no non-default SMB shares
   were discovered. Further authenticated enumeration may reveal accessible
   file shares or GPO-mounted content in SYSVOL/NETLOGON.

6. Web Application Hiring Portal: The PHP 8.2.12 application on port 80 may
   contain additional vulnerabilities (Local File Inclusion, Remote Code
   Execution, SQL injection) beyond the ZIP upload vector.

7. Service Account Credential Theft: Further investigation of web_svc and
   monitoring_svc may reveal Kerberos delegation settings, stored credentials,
   or DPAPI-protected secrets.

8. BloodHound Full Path Analysis: Additional attack paths in BloodHound may
   exist from monitoring_svc or other owned objects that were not identified
   in the initial analysis.

9. DCSync Attack: If monitoring_svc or any owned account gains replication
   rights (DS-Get-Changes, DS-Get-Changes-All), the entire domain can be
   compromised via secretsdump.py DCSync.

10. monitoring_svc Privilege Analysis: The account has SeMachineAccountPrivilege
    (Add workstations to domain) which could be abused to create a new computer
    object and perform RBCD (Resource-Based Constrained Delegation) attacks.

================================================================================
                        7. RECOMMENDATIONS
================================================================================

+----+----------+----------------------------------------------------+--------+
| #  | Priority | Recommendation                                     | Effort |
+----+----------+----------------------------------------------------+--------+
|  1 | CRITICAL | Patch CVE-2025-24071 on all Windows systems        | Medium |
|  2 | CRITICAL | Disable SMB null session authentication             | Low    |
|  3 | CRITICAL | Audit and remediate dangerous ACLs (AddSelf,       | Medium |
|    |          | ForceChangePassword) on IT_SUPPORT and groups      |        |
|  4 | CRITICAL | Enforce strong service account passwords; migrate  | Medium |
|    |          | to gMSA where possible                              |        |
|  5 | CRITICAL | Patch CVE-2024-0670 by updating CheckMK Agent      | Low    |
|  6 | HIGH     | Implement tiered administration model (TIER 0/1/2) | High   |
|  7 | HIGH     | Restrict ZIP upload functionality to sandboxed     | Medium |
|    |          | processing or disable entirely if not required     |        |
|  8 | HIGH     | Add SMB signing and EPA to prevent NTLM relay      | Low    |
|  9 | MEDIUM   | Enforce NTP time synchronization across all        | Low    |
|    |          | domain systems                                      |        |
| 10 | MEDIUM   | Review and harden domain password policies         | Low    |
| 11 | MEDIUM   | Monitor Security Event Log for ACL abuse            | Low    |
|    |          | indicators (Event IDs 4728, 4732, 4756)            |        |
| 12 | LOW      | Remove self-service group membership capabilities  | Low    |
|    |          | from all non-administrative accounts               |        |
+----+----------+----------------------------------------------------+--------+

================================================================================
                        8. CONCLUSION
================================================================================

The nanocorp.htb Active Directory environment was found to have multiple
security weaknesses enabling initial access, privilege escalation, and
domain controller shell access. The most critical findings are the combination
of CVE-2025-24071 exploitation leading to credential exposure, and the
dangerous ACL relationships allowing service accounts to escalate privileges
through group membership abuse and password reset attacks.

The engagement achieved:
  - Initial domain access via cracked web_svc credentials
  - Privilege escalation via AddSelf abuse on IT_SUPPORT group
  - Password reset of monitoring_svc via ForceChangePassword ACL
  - Kerberos TGT acquisition for monitoring_svc
  - Remote shell access to dc01.nanocorp.htb via Evil-WinRM
  - User flag captured from monitoring_svc desktop
  - CVE-2024-0670 CheckMK LPE identified but not fully exploited

The complete attack chain from unauthenticated network access to domain
controller shell access was demonstrated. Further escalation to Domain
Administrator may be possible through CVE-2024-0670 exploitation or
additional ACL abuse from the monitoring_svc account.

Remediation should prioritize patching CVE-2025-24071, eliminating null
session authentication, auditing and hardening Active Directory ACLs, and
migrating service accounts to managed accounts with automatic password
rotation.

================================================================================
                        9. LESSONS LEARNED
================================================================================

This machine reinforced several important lessons for Active Directory
penetration testing:

1. Enumeration over Exploitation: The real challenge was not a complicated
   exploit but identifying the correct path through careful enumeration.
   BloodHound identified the path, but understanding the underlying AD
   permissions was critical to executing it.

2. Kerberos Time Synchronization: Clock skew issues (~7 hours difference)
   completely blocked Kerberos authentication progress. Always verify NTP
   synchronization before attempting Kerberos-based attacks:
     sudo timedatectl set-ntp false && sudo ntpdate TARGET_IP

3. Active Directory Delegation: Privilege escalation in AD environments often
   relies on delegated permissions (AddSelf, ForceChangePassword, WriteDACL)
   rather than traditional software vulnerabilities. Understanding ACL chains
   between groups and users is essential.

4. Manual Enumeration Matters: While automated tools (BloodHound, WinPEAS)
   are invaluable, manual PowerShell enumeration frequently reveals findings
   that automated tools miss. The CheckMK Agent was identified as a
   non-Microsoft service through manual inspection rather than automated
   scanning.

5. File Permission Context: The CVE-2024-0670 exploit required careful
   handling of file paths and execution context. The seed files MUST be in
   C:\Windows\Temp (not C:\ProgramData) and the execution context must have
   write access to that directory. Using RunasCs to switch to web_svc was
   necessary because monitoring_svc could not write to C:\Windows\Temp.

6. Machine State Persistence: HTB machines reset between IP changes. All ACL
   modifications, group memberships, and password resets performed on one
   instance do not carry over to a new instance. Always verify the current
   machine state before assuming previous progress exists.

7. krb5.conf Configuration: A misconfigured /etc/krb5.conf can cause
   "Cannot find KDC" errors that are unrelated to the target. The default
   MIT Kerberos config includes ATHENA.MIT.EDU as the default realm, which
   overrides custom configurations if duplicate sections exist. Always
   ensure clean krb5.conf with proper [libdefaults], [realms], and
   [domain_realm] sections.


================================================================================
                        10. ATTACK PATH VISUALIZATION
================================================================================

Complete attack chain from initial access to SYSTEM privilege escalation:

  Anonymous Network Access
          |
          v
  +-------------------+     +------------------+
  | Port Scan (53,80, | --> | LDAP Enumeration |
  | 88,139,389,5986)  |     | (Domain: nanocorp|
  +-------------------+     | SID discovered)  |
          |                 +------------------+
          v                          |
  +-------------------+              v
  | SMB Null Auth     |     +------------------+
  | (domain/OS info)  |     | Web Enumeration  |
  +-------------------+     | (hiring portal,   |
          |                 |  ZIP upload)      |
          v                 +------------------+
  +-------------------+              |
  | Hiring Portal     |              v
  | ZIP Upload        |     +------------------+
  +-------------------+     | CVE-2025-24071   |
          |                 | (malicious ZIP    |
          v                 |  + Responder)     |
  +-------------------+     +------------------+
  | NTLMv2 Hash       |
  | Captured          |
  | NANOCORP\web_svc  |
  +-------------------+
          |
          v
  +-------------------+
  | John the Ripper   |
  | web_svc:          |
  | dksehdgh712!@#    |
  +-------------------+
          |
          v
  +-------------------+     +------------------+
  | Kerberos TGT      |     | LDAP Enum        |
  | via getTGT.py     |     | (users, groups,  |
  +-------------------+     |  computers)      |
          |                 +------------------+
          v                          |
  +-------------------+              v
  | BloodHound        |     +------------------+
  | Analysis          |     | ACL Chain        |
  | WEB_SVC -->       |     | Identified:      |
  | IT_SUPPORT -->    |     | AddSelf --> FCP  |
  | MONITORING_SVC    |     +------------------+
  +-------------------+
          |
          v
  +-------------------+
  | bloodyAD          |
  | AddSelf abuse:    |
  | web_svc -->       |
  | IT_SUPPORT        |
  +-------------------+
          |
          v
  +-------------------+
  | bloodyAD          |
  | ForceChangePass:  |
  | monitoring_svc    |
  | --> P@ssw0rd444!  |
  +-------------------+
          |
          v
  +-------------------+
  | getTGT.py         |
  | monitoring_svc    |
  | TGT obtained      |
  +-------------------+
          |
          v
  +-------------------+
  | Evil-WinRM        |
  | SSL (port 5986)   |
  | Kerberos auth     |
  | Shell: monitoring |
  | _svc@DC01         |
  +-------------------+
          |
          v
  +-------------------+
  | USER FLAG         |
  | C:\Users\monitor  |
  | ing_svc\Desktop\  |
  | user.txt          |
  +-------------------+
          |
          v
  +-------------------+
  | Local Enum        |
  | (WinPEAS/Manual)  |
  | CheckMK Agent     |
  | identified        |
  +-------------------+
          |
          v
  +-------------------+
  | CVE-2024-0670     |
  | MSI Repair LPE    |
  | (RunasCs +        |
  |  exploit.ps1)     |
  +-------------------+
          |
          v
  +-------------------+
  | SYSTEM SHELL      |
  | (reverse shell    |
  |  via nc.exe)      |
  +-------------------+
          |
          v
  +-------------------+
  | ROOT FLAG         |
  | C:\Users\Adminis  |
  | trator\Desktop\   |
  | root.txt          |
  +-------------------+


================================================================================
                                END OF REPORT
================================================================================
