+++
title = "Hack The Box: Luke"
date = "2019-09-13"
author = ""
cover = ""
tags = ["HTB"]
keywords = ["", ""]
description = ""
draft = true
showFullContent = false
+++

<!--more-->
{{< image src="/img/luke/luke_1.png" alt="Bastion Info Card" position="center" style="border-radius: 8px;" >}}

# Enumeration

As usual I'll start with a nmap syn scan to get open ports quickly.

```
nmap -sS --min-rate=1000 10.10.10.137
```


```
Nmap scan report for luke.htb (10.10.10.137)
Host is up (0.27s latency).
Not shown: 774 closed ports, 221 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
80/tcp   open  http
3000/tcp open  ppp
8000/tcp open  http-alt
```

Service detection scan:

```
nmap -sC -sV -p 21,22,80,3000,8000 -oA nmap/luke 10.10.10.137
```

```
Nmap scan report for luke.htb (10.10.10.137)
Host is up (0.24s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3+ (ext.1)
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0             512 Apr 14 12:35 webapp
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.33
|      Logged in as ftp
|      TYPE: ASCII
|      No session upload bandwidth limit
|      No session download bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3+ (ext.1) - secure, fast, stable
|_End of status
22/tcp   open  ssh?
80/tcp   open  http    Apache httpd 2.4.38 ((FreeBSD) PHP/7.3.3)
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.38 (FreeBSD) PHP/7.3.3
|_http-title: Luke
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (application/json; charset=utf-8).
8000/tcp open  http    Ajenti http control panel
|_http-title: Ajenti

```

## FTP

Based on the nmap results FTP allows anonymous login. Let's check it out.

```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        0             512 Apr 14 12:35 webapp
226 Directory send OK.
ftp> cd webapp
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-r-xr-xr-x    1 0        0             306 Apr 14 12:37 for_Chihiro.txt
```
There is a webapp folder and a txt file *for_Chihiro.txt* inside.

After downloading the file it says:

```
Dear Chihiro !!

As you told me that you wanted to learn Web Development and Frontend, I can give you a little push by showing the sources of
the actual website I've created .
Normally you should know where to look but hurry up because I will delete them soon because of our security policies !

Derry
```

At this point I have two possible usernames: *Chihiro* and *Jerry*, I have put this in my notes for later.

## HTTP

Now let's check out the web server. Loading the ip on the web browser reveals a simple web page with nothing interesting. Let's do some content discovery.

{{< image src="/img/luke/luke_2.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

### Content Discovery

For this we could use any tool to discover directories and interesting files. I will use dirsearch and dirbuster's medium wordlist. By default dirsearch will just get directories, the -f flag will force it to append file extensions that we provide to each entry on the wordlist.

```
dirsearch.py -f -u http://10.10.10.137/ -e html,txt,php -t 50 -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
```

{{< image src="/img/luke/luke_3.png" alt="backup share" position="center" style="border-radius: 8px;" >}}


```
$dbHost = 'localhost';
$dbUsername = 'root';
$dbPassword  = 'Zk6heYCyv6ZE9Xcg';
$db = "login";

$conn = new mysqli($dbHost, $dbUsername, $dbPassword,$db) or die("Connect failed: %s\n". $conn -> error);
```
