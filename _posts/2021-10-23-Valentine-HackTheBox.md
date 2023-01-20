---
title: Valentine - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-23 12:00:00 
categories: [HackTheBox,Linux]
tags: [heart bleed,dirtycow]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Valentine/00-banner.png
  width: 592
  height: 375
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Valentine` a retired linux machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console 
$ cat nmap-scan 
# Nmap 7.91 scan initiated Fri Oct 22 18:58:30 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.1.190
Increasing send delay for 10.129.1.190 from 0 to 5 due to 105 out of 349 dropped probes since last increase.
Nmap scan report for 10.129.1.190
Host is up (0.12s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 96:4c:51:42:3c:ba:22:49:20:4d:3e:ec:90:cc:fd:0e (DSA)
|   2048 46:bf:1f:cc:92:4f:1d:a0:42:b3:d2:16:a8:58:31:33 (RSA)
|_  256 e6:2b:25:19:cb:7e:54:cb:0a:b9:ac:16:98:c6:7d:a9 (ECDSA)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
| Issuer: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2018-02-06T00:45:25
| Not valid after:  2019-02-06T00:45:25
| MD5:   a413 c4f0 b145 2154 fb54 b2de c7a9 809d
|_SHA-1: 2303 80da 60e7 bde7 2ba6 76dd 5214 3c3c 6f53 01b1
|_ssl-date: 2021-10-22T23:59:15+00:00; +1s from scanner time.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Oct 22 18:59:15 2021 -- 1 IP address (1 host up) scanned in 44.78 seconds
```
Three ports are open `22:SSH`,`80:HTTP`,`443:SSL\HTTP`
The services of the machine are very outdated. We can run a nmap scan to discover if any of this services are vulnerable.

![01-nmap](/assets/img/HTB/Valentine/01-nmap.png)

The 443 port is vulnerable to `Heartbleed` 

## 80 Port
The web running on the 80 port shows a woman screaming and a heart bleeding, that's why I decided use `gobuster` to find new directories.

![02-gobuster](/assets/img/HTB/Valentine/02-gobuster.png)

The result show us some directories like `/encode` or `/decode` but the most interesting is `/dev` which contains two files, one of them is a string that is hex encoded.

![03-dev](/assets/img/HTB/Valentine/03-dev.png)

Using `cyberchef` we are able to decode retrieving a `SSH` private key

![04-decode](/assets/img/HTB/Valentine/04-decode.png)

To use it we need a password, I tried crack it using `ssh2john` but it didn't find the password.

## 443 Port
With this [script](https://gist.github.com/eelsivart/10174134) we can exploit `Hearbleed` against the victim machine.
```console
$ python heartbleed.py -n 100 10.129.1.190
```
After some tries running the script I finally recived the following strings.

![05-string](/assets/img/HTB/Valentine/05-string.png)

Decode with `base64`, we get the next: `heartbleedbelievethehype`

![06-password](/assets/img/HTB/Valentine/06-password.png)

This string could be the password of the ssh private key. 

![07-ssh](/assets/img/HTB/Valentine/07-ssh.png)

Voila!! We got shell as user hype.

# Privilage Escalation
## DirtyCow
Enumerating the box, it seems the machine is vulnerable to `dirtycow`

![09-uname](/assets/img/HTB/Valentine/09-uname.png)

And to exploit it I used this [variant](https://github.com/FireFart/dirtycow/blob/master/dirty.c).

![10-dirty](/assets/img/HTB/Valentine/10-dirty.png)

After execute it just access like the user `firefart` with the password `password`.

![11-firefart](/assets/img/HTB/Valentine/11-firefart.png)

## Tmux session
Other way to escalate privileges is [hijacking](https://int0x33.medium.com/day-69-hijacking-tmux-sessions-2-priv-esc-f05893c4ded0) the root's tmux session.

With `ps -faux` we can see all the proccess running in the machine 

![12-ps](/assets/img/HTB/Valentine/12-ps.png)

Is there when appears that root has a tmux session. Attach this tmux shell using the next command.
```text
tmux -S /.devs/dev_sess
```

![13-root](/assets/img/HTB/Valentine/13-root.png)

That’s it for now guys. Until next time.
# Resources
- https://github.com/dirtycow/dirtycow.github.io/wiki/PoCs
- https://int0x33.medium.com/day-69-hijacking-tmux-sessions-2-priv-esc-f05893c4ded0
