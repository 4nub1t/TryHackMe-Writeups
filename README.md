# TryHackMe Writeups

## Description

This repository contains my TryHackMe writeups as part of my learning path towards CPTS and OSCP.  
Each writeup follows a practical penetration testing methodology: recon, enumeration, exploitation, privilege escalation and reporting.

---

## Skills demonstrated

- Network and web application enumeration (nmap, gobuster, ffuf, whatweb)
- Web application exploitation (auth bypass, SQL injection, XSS, file upload, command injection)
- Linux and Windows privilege escalation
- Basic Active Directory and lateral movement (where applicable)
- Documentation and professional-style reporting

---

## Machines / Labs Index

| Machine / Lab                                                 | Category        | Difficulty | Key techniques / vulns                                   |
|---------------------------------------------------------------|-----------------|-----------|----------------------------------------------------------|
| [Blue](Blue/writeup.md)                                      | Windows         | Easy      | SMB enumeration, MS17-010 (EternalBlue), RCE, hashdump, NTLM cracking |
| [NetSec Challenge](Net-Sec-Challenge/writeup.md)              | Network / Linux | Easy      | Port scanning, service enumeration, privilege escalation |
| [Vulnerability Capstone](Vulnerability-Capstone/writeup.md)   | Web / Windows   | Medium    | SQLi, file upload, misconfiguration, privilege escalation |
| [Simple CTF](Simple-CTF/writeup.md)                           | Web / Linux     | Easy      | Directory brute-force, LFI, basic privilege escalation   |

*(Adjust categories, difficulty and techniques as needed.)*

---

## Repository structure

Each folder corresponds to a TryHackMe lab or machine and contains:

- `writeup.md` → step-by-step explanation of the challenge (methodology, commands, analysis)  
- `screenshots/` → relevant screenshots  

---

## How to use this repository

- Try to solve each room by yourself before reading the writeup.  
- Use the notes to understand the methodology, not to copy-paste commands blindly.  
- Feel free to fork the repo and adapt the structure to your own learning path.  

---

## Ethics note

> ⚠️ These writeups are for educational and personal purposes only.  
> They must not be used for illegal activities or to violate TryHackMe's terms of service.  
> Do not share flags or full solutions where it is against the rules of the platform.
