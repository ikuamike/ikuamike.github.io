+++
title = "SheHacksKE HackFest 2021 CTF WriteUp"
date = "2021-11-11"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["CTF", "Forensics"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
images = ["/img/shehacks-hackfest-2021/banner.png"]
+++

<!--more-->
{{< image src="/img/shehacks-hackfest-2021/scoreboard.png" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction

[@SheHacksKE](https://twitter.com/shehacks_ke) held their yearly hackfest event in October 2021 but it was online this year. The event included a CTF 
that was facilitated by [@eKRAALhub](https://twitter.com/eKRAALhub). [@mystic_kev](https://twitter.com/mystic_kev) and I participated in the CTF as
NoPwnNoGain and won it by completed all challenges. This writeup is for the challenges I was able to solve.

## Challenges: Firmware

### 1. Camera Kernel

{{< image src="/img/shehacks-hackfest-2021/kernel1.png" alt="" position="center" style="border-radius: 8px;" >}}

The provided file when extracted contains a bin file that is a linux image.

{{< image src="/img/shehacks-hackfest-2021/kernel2.png" alt="" position="center" style="border-radius: 8px;" >}}

By using binwalk to analyse the file we see the different file sections. 

```sh
binwalk demo.bin
```

The kernel starts at the second section indicated as the **OS Kernel Image** and includes the **lzma compressed 
data** to end at the beginning of the squashfs filesystem.

{{< image src="/img/shehacks-hackfest-2021/kernel3.png" alt="" position="center" style="border-radius: 8px;" >}}

To provide the flag in decimal we minus the header that identifies the firmware image.

```
2097216 - 64 = 2097152
```

Flag: Hackfest{2097152}

### 2. Backups

{{< image src="/img/shehacks-hackfest-2021/kernel4.png" alt="" position="center" style="border-radius: 8px;" >}}

Based on the binwalk output we can take the root filesystem as the first squashfs filesystem. We can calculate the size as below.

```
5570624 - 2097216 = 3473408
```

To extract it we can use **dd** command line tool then mount the filesystem.

```sh
dd if=demo.bin skip=2097216 count=3473408 bs=1 of=squashfs1 
```
```sh
sudo mount -t squashfs squashfs1 mount1
``` 
{{< image src="/img/shehacks-hackfest-2021/kernel5.png" alt="" position="center" style="border-radius: 8px;" >}}

On mounting the filesystem we see the backup folders.

Flag: Hackfest{backupa,backupd,backupk}

### 3. Semiconductor

{{< image src="/img/shehacks-hackfest-2021/kernel6.png" alt="" position="center" style="border-radius: 8px;" >}}

By looking around the root filesystem we can read the hostname of the device this image was retrieved from.

{{< image src="/img/shehacks-hackfest-2021/kernel7.png" alt="" position="center" style="border-radius: 8px;" >}}

Searching this name on google shows this is a semiconductor company confirming this is the flag.

{{< image src="/img/shehacks-hackfest-2021/kernel8.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: Hackfest{Ingenic}

### 4. Leaky Developer

{{< image src="/img/shehacks-hackfest-2021/kernel12.png" alt="" position="center" style="border-radius: 8px;" >}}

The rest of the filesystem is inside the jffs2 filesystem that starts offset **6225984** until the end.

{{< image src="/img/shehacks-hackfest-2021/kernel13.png" alt="" position="center" style="border-radius: 8px;" >}}

To extract it we can carve it out using dd.

```sh
dd if=demo.bin skip=6225984 bs=1 of=jffs
```
To mount the jffs2 file system, I found the below script to be useful.

```sh
#!/bin/sh
set -e
if [ -z "$1" ]; then
    echo usage: $0 jffs2_image directory
    exit
fi

if [ -z "$2" ]; then
    echo usage: $0 jffs2_image directory
    exit
fi

# modprobe mtdcore
modprobe jffs2

SIZE=$(du -h -k $1 | cut -f 1)
SIZE=8192 # 65536
ESIZE=$(( $SIZE / 6))
BUFS=512
modprobe mtdram \
 total_size=$SIZE \
 erase_size=${ESIZE} \
# writebuf_size=$BUFS

# modprobe mtdchar
modprobe mtdblock

sleep .25

dd if=$1 of=/dev/mtd0

mount -t jffs2 /dev/mtdblock0 $2
```
Once we mount the filesystem as below, there's a mount.sh file inside the bin directory that shows the folder in the
home directory.

{{< image src="/img/shehacks-hackfest-2021/kernel14.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: Hackfest{xuxuequan}

### 5. Root

{{< image src="/img/shehacks-hackfest-2021/kernel10.png" alt="" position="center" style="border-radius: 8px;" >}}

The shadow file in the root filesystem contains the root hash.

{{< image src="/img/shehacks-hackfest-2021/kernel9.png" alt="" position="center" style="border-radius: 8px;" >}}

I tried to crack this hash using the usual rockyou wordlist but got nothing so I jumped to google to see if I get any
hits.

{{< image src="/img/shehacks-hackfest-2021/kernel11.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: Hackfest{ismart12}

### 6. Extras

The above solutions were after some analysis of how I believe the challenges we meant to be solved. Initially I did the 
basic option of extracting the image using binwalk.

```sh
binwalk -e demo.bin
```
After this and looking through the resulting directories and some google searches later I found this post that allowed me to quickly answer
the questions.

[https://research.nccgroup.com/2020/07/31/lights-camera-hacked-an-insight-into-the-world-of-popular-ip-cameras/](https://research.nccgroup.com/2020/07/31/lights-camera-hacked-an-insight-into-the-world-of-popular-ip-cameras/)

### 7. The Rest

The rest of the challenges in this category were solved by Kelvin.

[https://github.com/mystickev/CTFwriteups/tree/Writeups/shehacks-2021](https://github.com/mystickev/CTFwriteups/tree/Writeups/shehacks-2021)

## Challenges: Protocols

### 1. Subscriber

{{< image src="/img/shehacks-hackfest-2021/kernel15.png" alt="" position="center" style="border-radius: 8px;" >}}

First thing I did was run nmap against the host and port to figure out what the service is running.

{{< image src="/img/shehacks-hackfest-2021/kernel16.png" alt="" position="center" style="border-radius: 8px;" >}}

On discovering this is mqtt, I downloaded [mqtt explorer](https://github.com/thomasnordquist/MQTT-Explorer/releases).

{{< image src="/img/shehacks-hackfest-2021/kernel17.png" alt="" position="center" style="border-radius: 8px;" >}}

Once we connect we start receiving messages, after a while the flag is sent as a message.

{{< image src="/img/shehacks-hackfest-2021/kernel18.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: Hackfest{Howdy Friend}

### 2. Hardcoded

{{< image src="/img/shehacks-hackfest-2021/kernel19.png" alt="" position="center" style="border-radius: 8px;" >}}

The provided zip file has only two files inside. 

{{< image src="/img/shehacks-hackfest-2021/kernel20.png" alt="" position="center" style="border-radius: 8px;" >}}

The config.php is somewhat encrypted.

Using this google search I found an article with the decryption method.

{{< image src="/img/shehacks-hackfest-2021/kernel21.png" alt="" position="center" style="border-radius: 8px;" >}}

[https://blog.securityevaluators.com/terramaster-nas-vulnerabilities-discovered-and-exploited-b8e5243e7a63](https://blog.securityevaluators.com/terramaster-nas-vulnerabilities-discovered-and-exploited-b8e5243e7a63)

By running the below command we can get the email from the config.php

```sh
openssl aes-256-cbc -d -K 3834326434326239383837366635383166306466626566623063643262356333 -iv 0 -in config.php > config.php.out
```

{{< image src="/img/shehacks-hackfest-2021/kernel22.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: Hackfest{kalcaddle@qq.com}

## Conclusion

This was a nice CTF with some not so common categories. Playing with dd was a learning process for me, I haven't used it much before.

Here are the challenge files incase the CTF site goes down.











