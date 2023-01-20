---
title: Irked - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-25 12:00:00 
categories: [HackTheBox,Linux]
tags: [exploit,suid]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Irked/00-banner.png
  width: 597
  height: 372
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Irked` a retired linux machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.

```console
$ nmap -p- -v --open -T5 10.129.1.108
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-25 14:59 -05
Initiating Ping Scan at 14:59

Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
111/tcp   open  rpcbind
8067/tcp  open  infi-async
65534/tcp open  unknown

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 44.59 seconds

$ nmap -sC -sV -p22,80,111,8067,65534 -oN nmap-scan 10.129.1.108
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-25 15:04 -05
Nmap scan report for 10.129.1.108
Host is up (0.13s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey:
|   1024 6a:5d:f5:bd:cf:83:78:b6:75:31:9b:dc:79:c5:fd:ad (DSA)
|   2048 75:2e:66:bf:b9:3c:cc:f7:7e:84:8a:8b:f0:81:02:33 (RSA)
|   256 c8:a3:a2:5e:34:9a:c4:9b:90:53:f7:50:bf:ea:25:3b (ECDSA)
|_  256 8d:1b:43:c7:d0:1a:4c:05:cf:82:ed:c1:01:63:a2:0c (ED25519)
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesn't have a title (text/html).
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          37878/udp   status
|   100024  1          46857/udp6  status
|   100024  1          53956/tcp   status
|_  100024  1          59587/tcp6  status
8067/tcp  open  irc     UnrealIRCd (Admin email djmardov@irked.htb)
65534/tcp open  irc     UnrealIRCd (Admin email djmardov@irked.htb)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.58 seconds
```

The open ports are: `22:SSH`,`80:HTTP`,`111:RPC` and `IRC:8067,65534`.

## HTTP Enumeration
Checking the web.

![01-web](/assets/img/HTB/Irked/01-web.png)

Not give us much information and gobuster only found `/manual` which is the default Apache HTTP server page.

![02-gobuster](/assets/img/HTB/Irked/02-gobuster.png)

# Initial Foothold
Searching about `UnrealIRCd` I came across with a [backdoor command execution.](
https://github.com/Ranger11Danger/UnrealIRCd-3.2.8.1-Backdoor/blob/master/exploit.py)

![03-backdoor](/assets/img/HTB/Irked/03-backdoor.png)

The exploit worked and we got a shell.
Using `find / -perm -4000 2>/dev/null` to show the SUID files, one of the many that we get is  `/usr/bin/viewuser`

![04-suid](/assets/img/HTB/Irked/04-suid.png)

and when we want execute it, it displays a error saying that a file named`listusers` not was found.

![05-viewer](/assets/img/HTB/Irked/05-viewer.png)

How the file doesn't exist, we can create it and insert into it a revese shell.

![06-file](/assets/img/HTB/Irked/06-file.png)

as soon as we run the SUID file we get a shell as root.

![07-root](/assets/img/HTB/Irked/07-root.png)

That’s it for now guys. Until next time.
