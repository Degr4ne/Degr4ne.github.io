---
title: OverPass3 - TryHackMe Walkthrough
author: Degr4ne
date: 2021-09-12 12:00:00 
categories: [TryHackMe]
tags: [gpg,nfs]
math: false
mermaid: false
image:
  src: /assets/img/THM/OverPass3/00-OverPass3.png
  width: 512
  height: 512
---
Hello guys, today we are gonna do a walkthrough on [OverPass 3](https://tryhackme.com/room/overpass3hosting) the box from TryHackMe, this box was rated medium. Let’s jump in.

We will start with `nmap` to know which ports are open and what services are running.
```console
$ nmap -v -sC -sV -oN overpass 10.10.40.242
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 de:5b:0e:b5:40:aa:43:4d:2a:83:31:14:20:77:9c:a1 (RSA)
|   256 f4:b5:a6:60:f4:d1:bf:e2:85:2e:2e:7e:5f:4c:ce:38 (ECDSA)
|_  256 29:e6:61:09:ed:8a:88:2b:55:74:f2:b7:33:ae:df:c8 (ED25519)
80/tcp open  http    Apache httpd 2.4.37 ((centos))
| http-methods: 
|   Supported Methods: POST OPTIONS HEAD GET TRACE
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.37 (centos)
|_http-title: Overpass Hosting
Service Info: OS: Unix
```

Looking at the nmap scan we can see three open ports: FTP, SSH and HTTP.
As you can see in the screenshot below FTP doesn't have Anonymous login allowed.

![01-ftp](/assets/img/THM/OverPass3/01-ftp.png)

We could try later a FTP login if we find some credencials but for now let's open the website.

![02-web](/assets/img/THM/OverPass3/02-web.png)

What we see on the website is the nick of the employees, this could help us in the future so we must remember them.

### Potencial users
- Paradox
- Elf 
- MuirlandOracle
- NinjaJc01

Looking at the source code of the web page, we find a comment but this isn't helpful.

![03-sourcecode](/assets/img/THM/OverPass3/03-sourcecode.png)

Enumerating with `gobuster`we found `/backups` directory that looks interesting.
```console
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.40.242/ -t 50 -o normal-fuzz
```
![04-gobuster](/assets/img/THM/OverPass3/04-gobuster.png)

Going to `/backups` directory we see that it contains a backup.zip.

![05-backups](/assets/img/THM/OverPass3/05-backups.png)

Once downloaded the ZIP file, we can extract its content using `7z x backup.zip`.

We get 2 files, `CustomerDetails.xlsx.gpg` that is a xlsx file encrypted with gpg and the other a private key used to decrypte the file. For know more about how gpg works you can search [here](https://www.gnupg.ordg/gph/en/manual.html).

![06-gpg](/assets/img/THM/OverPass3/06-gpg.png)

Let's see the content of the xlsx file using a online viewer.

![07-excelviewer](/assets/img/THM/OverPass3/07-excelviewer.png)

We can notice some credentials but what focused my attention was the username `paradox` and his password.

Username|Password
-|-
paradox | ShibesAreGreat123

Next I logged into FTP services. I realized that this FTP share access to the website's sources.

![08-ftp](/assets/img/THM/OverPass3/08-ftp.png)

With the idea of upload a reverse shell I copied into my local directory a php reverse shell template using `cp /usr/share/webshells/php/php-reverse-shell.php .`. After that, I renamed the file to `shell.php` and added my ip and listener port. 

![09-phpshell](/assets/img/THM/OverPass3/09-phpshell.png)

First I uploaded the file in FTP service using `put` command , Second I started a netcat  listener and for last I tried to access that reverse shell  from the website.

![10-RevShell](/assets/img/THM/OverPass3/10-RevShell.png)

Once we have a reverse shell, we see the user paradox, so let's connect as `paradox`.

![11-paradox](/assets/img/THM/OverPass3/11-paradox.png)

Before running [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS), I like to do a manual enumeration but this time I didn't found nothing, so let's upload linpeas to the box using python3 to share the tool and curl to download it.

![12-linpeas](/assets/img/THM/OverPass3/12-linpeas.png)

Checking the linpeas's output I found that NFS service is running and sharing `/home/james/` directory with `no_root_squash`. Click [here](https://book.hacktricks.xyz/linux-unix/privilege-escalation/nfs-no_root_squash-misconfiguration-pe) for more information about this misconfiguration.

![13-nfs](/assets/img/THM/OverPass3/13-nfs.png)

But when I tried to scan the NFS port from my box I didn't found nothing, this NFS service must be only accessible to localhost.

For have access to NFS service we can use [chisel](https://github.com/jpillora/chisel/releases) a port forwarder.

![14-chisel](/assets/img/THM/OverPass3/14-chisel.png)

The commands used are:
```
A: ./chisel server --reverse --port 1234 
V: ./chisel client <your ip>:1234 R:2049:127.0.0.1:2049
```

After we have access to NFS services we can make a mount in our machine from `/hame/james` directory as screenshot below.

![15-mount](/assets/img/THM/OverPass3/15-mount.png)

Into `/.ssh` directory we can see a id_rsa file, saving this private SSH key and changing his permission to `600` we can loggin into the box using it, as it displays in the screenshot below.

![16-idrsa](/assets/img/THM/OverPass3/16-idrsa.png)

Like `/hame/james` is configured as **no_root_squash**, then we can write inside that directory as if you were the local root of the machine. So let's upload a bash shell as root user. And changed his permission to SUID using `chmod` command.

![17-bash](/assets/img/THM/OverPass3/17-bash.png)

Now, the binary is with the SUID permission.

![18-ls](/assets/img/THM/OverPass3/18-ls.png)

For be root we have to exec the binary adding `-p` argument.

![19-suid](/assets/img/THM/OverPass3/19-suid.png)

For unmount the /home/james in our mount directory we use `sudo umount ./mount`.

That’s it for now guys. Until next time.
