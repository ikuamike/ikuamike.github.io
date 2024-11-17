+++
title = "Vulnhub: Symfonos 2"
date = "2021-02-10"
author = ""
tags = ["Intermediate", "Vulnhub", "OSCP Prep", "Samba","ProFTPD", "CVE-2015-3306","LibreNMS","CVE-2018-20434", "Sudo", "MySQL"]
keywords = ["", ""]
description = ""
showFullContent = false
cover = "/img/symfonos1/symfonos1.png"
images = ["/img/symfonos1/symfonos1.png"]
+++



| Difficulty | Release Date | Author | 
| ---------- | ------------ | ------ | 
| Intermediate | 18 July 2019   | Zayotic |

# Summary

This box had quite a good number of misconfigurations and vulnerabilities. Initial access was through copying a shadow
backup file to a smb share accessible anonymously using a file copy vulnerability in proftpd. Then lateral movement 
and privilege escalation was achieved by exploiting rce a locally running librenms instance and finally abusing
sudo permissions on mysql to get root.

# Reconnaissance

```
Nmap scan report for 192.168.191.129
Host is up (0.00028s latency).

PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         ProFTPD 1.3.5
22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey:
|   2048 9d:f8:5f:87:20:e5:8c:fa:68:47:7d:71:62:08:ad:b9 (RSA)
|   256 04:2a:bb:06:56:ea:d1:93:1c:d2:78:0a:00:46:9d:85 (ECDSA)
|_  256 28:ad:ac:dc:7e:2a:1c:f6:4c:6b:47:f2:d6:22:5b:52 (ED25519)
80/tcp  open  http        WebFS httpd 1.21
|_http-server-header: webfs/1.21
|_http-title: Site doesn't have a title (text/html).
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
Service Info: Host: SYMFONOS2; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h00m00s, deviation: 3h27m51s, median: 0s
|_nbstat: NetBIOS name: SYMFONOS2, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.5.16-Debian)
|   Computer name: symfonos2
|   NetBIOS computer name: SYMFONOS2\x00
|   Domain name: \x00
|   FQDN: symfonos2
|_  System time: 2021-01-25T07:02:21-06:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-01-25T13:02:21
|_  start_date: N/A
```

# Enumeration

## Samba (139/445)

Running smbmap on the target we see we can access the **anonymous** share without credentials:

```sh
smbmap -R -H 192.168.191.129
```

{{< image src="/img/symfonos2/smbmap.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Using smbclient to download log.txt from the anonymous share.

```sh
smbclient \\\\192.168.191.129\\anonymous -N -c 'get backups\log.txt log.txt'
```
From log.txt, we discover a backup of the shadow file is created in /var/backups/

{{< image src="/img/symfonos2/symfonos2-1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

## FTP (21)

Checking for known vulnerabilities of the proftp version using searchsploit, we find several.

{{< image src="/img/symfonos2/symfonos2-2.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The command execution exploits available rely on creating a php file on the existing webserver that would be executed.
However, the web server is running [webfs](https://github.com/ourway/webfsd) which only serves static content.

We can only take advantage of the file copy vulnerability. From log.txt, the smb configuration gives the location
of the anonymous share:

{{< image src="/img/symfonos2/symfonos2-3.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We can exfiltrate information to the anonymous share by exploiting file copy on proftpd. Using the PoC shown here 
https://www.exploit-db.com/exploits/36742, let's get the shadow.bak file. You can either use netcat or telnet. 
In this case I use netcat.

```sh
nc -vn 192.168.191.129 21
```

{{< image src="/img/symfonos2/symfonos2-4.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Running smbmap command again we see that we have shadow.bak file.

{{< image src="/img/symfonos2/symfonos2-5.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

## Shell as aeolus

After downloading the shadow file from the smb we can crack it with john and rockyou wordlist.

```sh
john --wordlist=/usr/share/wordlists/rockyou.txt shadow.bak
```

{{< image src="/img/symfonos2/symfonos2-6.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Using sergioteamo as the password we can access ssh as aeolus.

Doing some basic manual enumeration, we see some ports listening on localhost. 

{{< image src="/img/symfonos2/symfonos2-7.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

8080 would most likely be a webserver, so I'll tunnel to it first using an ssh port forward.

```sh
ssh -L 8000:127.0.0.1:8080 aeolus@192.168.191.129
```
When we visit http://127.0.0.1:8000 on the browser, we find that LibreNMS is running here.

{{< image src="/img/symfonos2/symfonos2-8.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Using searchsploit again we find two existing vulnerabilities for version 1.46. 

{{< image src="/img/symfonos2/symfonos2-13.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Since we can't verify the version of librenms currently running, we'll try the exploits and see if they work. I'll go for the RCE.


Using the exploit script provided here https://www.exploit-db.com/exploits/47044, we need to provide url, session cookies,
rhost and rport where the reverse shell will be received.

Login using the earlier discovered credentials **aeolus : sergioteamo**, then retrieve the cookies using burpsuite.

{{< image src="/img/symfonos2/symfonos2-10.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Run the exploit as below and catch the reverse shell using netcat.

{{< image src="/img/symfonos2/symfonos2-11.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We are now the cronus user. 

## Getting a root shell

Running sudo -l reveals cronus can run mysql as root with no password. 

Dropping into mysql as root user, we can get a root shell using the below command.

```sh
\! bash
```
{{< image src="/img/symfonos2/symfonos2-12.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

You can reach/follow me on Twitter if you have some feedback or questions.

Twitter: [ikuamike](https://twitter.com/ikuamike)

Github: [ikuamike](https://github.com/ikuamike)
