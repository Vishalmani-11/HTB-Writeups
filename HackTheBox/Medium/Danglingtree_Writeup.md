# DanglingTree HTB-Box Writeup 
```text
DANGLINGTREE – COMPLETE TECHNICAL PENETRATION TEST / HTB REPORT

Assessment Type: Hack The Box / Authorized CTF Lab
Target: danglingtree.htb
Domain: danglingtree.htb
Domain Controller IP: 10.129.9.189
Primary CA: danglingtree-DC-CA
Status: Administrator certificate authentication and NT hash recovery SUCCESS

1. EXECUTIVE SUMMARY

The DanglingTree Windows Active Directory environment was compromised
through a multi-stage attack chain:

Windows Admin Center command execution -> anderson.w -> internal
SmarterMail on 127.0.0.1:17017 -> Chisel forwarding -> SmarterMail
CVE-2026-24423 -> svc_mail -> SmarterMail backup / DPAPI artifacts ->
alex.o credentials -> SUPPORT-IT / ForceChangePassword -> jake.h -> AD
CS EmployeeAuthTemplate -> ESC1-style Administrator certificate ->
Administrator TGT and NT hash

The final recorded stage successfully recovered the Administrator NT
hash. The provided writeup documents using that hash to access the
Domain Controller and retrieve root.txt.

2. RECONNAISSANCE

Target: danglingtree.htb

Relevant services observed:
53 DNS
88 Kerberos
389 LDAP
445 SMB
636 LDAPS
5985 WinRM
6600 Windows Admin Center
17017 SmarterMail (localhost/internal)

The writeup also documents an anonymously accessible SMB IT share
containing DanglingTree_RoE_Assessment.pdf.

3. INITIAL FOOTHOLD – WINDOWS ADMIN CENTER

Windows Admin Center was exposed on TCP/6600.

The WAC API exposed: /api/WinREST/PowerShell/nodes/dc/invokeCommand

Manipulating the PowerShell execution request resulted in command
execution as:

danglingtree.w

This established the initial foothold.

4. INTERNAL SMARTERMAIL DISCOVERY

Local service enumeration identified:

127.0.0.1:17017

The port belonged to SmarterMail and was bound only to localhost.

Chisel was used to forward the internal service from the compromised
Windows host to the attack machine.

5. SMARTERMAIL RCE

The lab instance was vulnerable to CVE-2026-24423.

The writeup identifies: /api/v1/settings/sysadmin/connect-to-hub

as the vulnerable endpoint.

Successful exploitation resulted in command execution as:

danglingtree_mail

6. SMARTERMAIL BACKUP DISCOVERY

The SmarterMail backup was located under:

C:\SmarterMail\Domains\danglingtree.htb.bak

The backup contained accounts including:

amelia.r
emma.s
liam.m
noah.b
oliver.t
sophia.k
svc_mail

The noah.b directory contained:

FileStore
Mail
acquaintances.sbin
folders.json
settings.json

The noah.b settings identified the account as:

noah.b@danglingtree.htb

7. DPAPI RECOVERY

Recovered artifacts:

Master Key: f53fcaba-f057-48e8-8f92-0180d274bf0f
Credential Blob: 57FFB67D684C67F09E7153B9C7CC3940

The master key was decrypted with:

Password: RiverDragon#Storm25
SID: S-1-5-21-4220238332-57023728-1129110646-1602

Recovered master key:

7120d9adb3b8ccd8901bf9e2a29afabcbbcbdb5a13a24a1817bda49097c7ff3c8e5d71f34ae43850a136dc64dbd37061d4f9c34bdbdca21aa8af57d26baad0d8

The credential blob decrypted successfully and produced:

Username: alex.o
Password: SunsetMountainPeak@2025
Target: Domain:target=PC01.danglingtree.htb

8. ACTIVE DIRECTORY ENUMERATION

BloodHound analysis identified:

alex.o -> SUPPORT-IT -> ForceChangePassword -> jake.h

This allowed the lab password for jake.h to be changed.

9. JAKE.H

Jake's working lab password became:

NewStrongPassword!2026

Jake was then successfully authenticated.

Relevant membership discovered for Jake included:

Helpdesk_Cert_Support
Template_Editors
DevOps_PKI

10. AD CS ENUMERATION

Certificate Authority:

danglingtree-DC-CA

CA host:

dc.danglingtree.htb

Enabled templates included:

RemoteAccessVPN
EmployeeAuthTemplate
VPNUserTemplate
DirectoryEmailReplication
DomainControllerAuthentication
KerberosAuthentication
EFSRecovery
EFS
DomainController
WebServer
Machine
User
SubCA
Administrator

Certipy also identified an ESC7-related dangerous CA permission
condition associated with Helpdesk_Cert_Support.

11. EMPLOYEEAUTHTEMPLATE

The CA published EmployeeAuthTemplate, but an LDAP search initially
returned no corresponding template object.

The writeup referenced the external artifact:

artifacts/esc1_template.ldif

The actual artifact was not present in the supplied files, so the
template was reconstructed in the lab from the documented requirements.

DN:

CN=EmployeeAuthTemplate,CN=Certificate Templates,CN=Public Key Services,
CN=Services,CN=Configuration,DC=danglingtree,DC=htb

Important attributes:

objectClass: pKICertificateTemplate
cn: EmployeeAuthTemplate
name: EmployeeAuthTemplate
displayName: EmployeeAuthTemplate
msPKI-Cert-Template-OID: 1.3.6.1.4.1.311.21.999
msPKI-Certificate-Name-Flag: 1
msPKI-Minimal-Key-Size: 2048
msPKI-Template-Schema-Version: 2
pKIExtendedKeyUsage: 1.3.6.1.5.5.7.3.2
pKIKeyUsage: 0xA0
pKIDefaultKeySpec: 1

LDAP subsequently confirmed one matching entry.

12. TEMPLATE ACL

Authenticated Users:

S-1-5-11

were granted GenericAll over EmployeeAuthTemplate.

bloodyAD confirmed:

S-1-5-11 has now GenericAll on EmployeeAuthTemplate

Certipy then reported:

Enabled: True
Client Authentication: True
Enrollee Supplies Subject: True
Requires Manager Approval: False
Authorized Signatures Required: 0

13. TEMPLATE CORRECTION

The first certificate request failed with:

0x80094800 CERTSRV_E_UNSUPPORTED_CERT_TYPE

Comparison against the built-in User template showed the custom object
lacked several CA-compatible attributes, including:

flags
revision
pKICriticalExtensions
pKIExpirationPeriod
pKIOverlapPeriod
pKIDefaultCSPs
msPKI-Template-Minor-Revision

After correction, Certipy reported:

Enabled: True
Client Authentication: True
Enrollee Supplies Subject: True
Enrollment Flag: IncludeSymmetricAlgorithms PublishToDs
AutoEnrollment
Extended Key Usage: Client Authentication
Requires Manager Approval: False
Authorized Signatures Required: 0
Schema Version: 1
Validity Period: 1 year
Renewal Period: 6 weeks
Minimum RSA Key Length: 2048

14. ADMINISTRATOR CERTIFICATE REQUEST

The first request after correction hit an RPC/NETBIOS timeout.

A debug retry connected successfully to:

ncacn_np:10.129.9.189[]

Certipy then reported:

Request ID is 19
Successfully requested certificate
Got certificate with UPN 'administrator@danglingtree.htb'

Certificate identity:

UPN: administrator@danglingtree.htb
SID: S-1-5-21-4220238332-57023728-1129110646-500

Output:

esc1_admin.pfx

15. KERBEROS CLOCK SKEW

The first certificate authentication attempt failed with:

KRB_AP_ERR_SKEW Clock skew too great

NTP synchronization against the Domain Controller was performed.

The observed offset was approximately:

+25561.9 seconds

After synchronization, certificate authentication succeeded.

16. CERTIFICATE AUTHENTICATION

Command used:

certipy auth -pfx esc1_admin.pfx -dc-ip 10.129.9.189 -domain danglingtree.htb

Certipy reported:

SAN UPN: administrator@danglingtree.htb
SAN URL SID: S-1-5-21-4220238332-57023728-1129110646-500
Security Extension SID: S-1-5-21-4220238332-57023728-1129110646-500

Then:

Got TGT
Saving credential cache to administrator.ccache
Wrote credential cache to administrator.ccache
Trying to retrieve NT hash for 'administrator'
Got hash for 'administrator@danglingtree.htb'

Recovered Administrator credential material:

NTLM: aad3b435b51404eeaad3b435b51404ee:8cacb3a97e460c65d105ca7cd9913925

NT hash:

8cacb3a97e460c65d105ca7cd9913925

17. FINAL OBJECTIVE

The recorded evidence ends after successful Administrator certificate
authentication and NT hash recovery.

The supplied writeup identifies the final objective as accessing the
Domain Controller as Administrator and reading:

C:\Users\Administrator\Desktop\root.txt

The root flag itself was not included in the material provided for this
report.

18. ARTIFACTS COLLECTED

SmarterMail.Standard.dll
f53fcaba-f057-48e8-8f92-0180d274bf0f
57FFB67D684C67F09E7153B9C7CC3940
esc1_template.ldif
esc1_admin.key
esc1_admin.pfx
administrator.ccache

SmarterMail.Standard.dll:
Size: 36,341,248 bytes
Type: PE32 executable for MS Windows
Architecture: Intel i386
Format: Mono/.NET assembly

Reverse engineering located:

SmarterMail.Standard.Utilities/CryptographyHelper.cs

The decompiled assembly exposed CryptographyHelper encryption/decryption
functionality and the keymap2 field used during investigation of
SmarterMail's encrypted credential handling.

19. SECURITY FINDINGS

FINDING 1 – Windows Admin Center command execution
Impact: Critical in the lab
WAC permitted PowerShell command execution that produced the initial foothold.

FINDING 2 – Internal SmarterMail exposure
Impact: High
SmarterMail was reachable from the compromised host despite being localhost-only.

FINDING 3 – SmarterMail CVE-2026-24423
Impact: Critical
The vulnerable SmarterMail deployment permitted unauthenticated RCE.

FINDING 4 – Backup / DPAPI exposure
Impact: Critical
Application backup data exposed artifacts that enabled credential recovery.

FINDING 5 – Excessive ForceChangePassword privilege
Impact: High
alex.o could change jake.h's password through the SUPPORT-IT relationship.

FINDING 6 – Excessive AD CS template control
Impact: Critical
Jake's rights permitted creation/modification of a certificate template that could authenticate users and accept an enrollee-supplied subject.

FINDING 7 – ESC7-related CA permissions
Impact: High
Certipy identified dangerous CA permissions associated with Helpdesk_Cert_Support.

20. RECOMMENDATIONS

Windows Admin Center:
- Restrict WAC to trusted administrative networks.
- Require strong authentication and least privilege.
- Audit WAC PowerShell/API execution.
- Monitor unusual WAC API requests.

SmarterMail:
- Patch vulnerable versions.
- Restrict administrative interfaces.
- Protect application backups.
- Prevent service accounts from reading unrelated user profiles.
- Rotate exposed credentials.

Active Directory:
- Review ForceChangePassword delegations.
- Minimize Template_Editors and similar privileged group membership.
- Regularly review BloodHound privilege paths.
- Apply least privilege to certificate-template administration.

AD CS:
- Audit certificate-template ACLs.
- Remove unnecessary GenericAll/Write permissions.
- Restrict who can create or modify certificate templates.
- Review enrollee-supplied subject/SAN capabilities.
- Review client-authentication EKUs.
- Audit CA permissions for ESC7-style privileges.
- Monitor certificates requested for privileged UPNs/SIDs.

Kerberos:
- Maintain reliable domain-wide time synchronization.
- Monitor excessive clock offsets.

21. TIMELINE

1. Enumerated danglingtree.htb.
2. Identified Windows Admin Center on TCP/6600.
3. Obtained anderson.w command execution.
4. Discovered localhost SmarterMail on TCP/17017.
5. Forwarded SmarterMail with Chisel.
6. Exploited CVE-2026-24423.
7. Obtained svc_mail execution.
8. Located the SmarterMail backup.
9. Located noah.b artifacts.
10. Recovered DPAPI master key and credential blob.
11. Recovered alex.o credentials.
12. Enumerated Active Directory.
13. Identified ForceChangePassword from alex.o to jake.h.
14. Changed Jake's password.
15. Enumerated AD CS.
16. Created EmployeeAuthTemplate in LDAP.
17. Granted S-1-5-11 GenericAll.
18. Corrected template configuration.
19. Requested Administrator certificate.
20. Saved esc1_admin.pfx.
21. Corrected Kerberos clock skew.
22. Authenticated using the certificate.
23. Obtained Administrator TGT.
24. Recovered Administrator NT hash.

22. FINAL STATUS

Initial foothold: SUCCESS
svc_mail compromise: SUCCESS
DPAPI recovery: SUCCESS
alex.o credential recovery: SUCCESS
jake.h compromise: SUCCESS
AD CS template creation: SUCCESS
Template ACL modification: SUCCESS
Administrator certificate: SUCCESS
Certificate authentication: SUCCESS
Administrator TGT: SUCCESS
Administrator NT hash: SUCCESS
root.txt retrieval: NOT RECORDED IN SUPPLIED EVIDENCE

SOURCE / EVIDENCE NOTE

The uploaded DanglingTree writeup was used as the reference for the
intended attack narrative and AD CS sequence. It explicitly references
artifacts/esc1_template.ldif, but that artifact was not present in the
supplied files. The report therefore distinguishes the writeup's
documented sequence from the template reconstruction performed during
the lab.

The source material documents the AD CS sequence of creating the
template, granting enrollment permissions, requesting an Administrator
certificate, and authenticating with the resulting PFX. See the supplied
writeup for that sequence.
```
