+++
title = "P3RF3CT R00T CTF 2024 Writeup - Forensics"
date = "2024-11-17T12:17:45+03:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["CTF", "Forensics", "Master File Table", "MFT", "MFTECmd", "MFT Explorer"]
keywords = []
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
images = ["/img/p3rf3ctr00t_ctf_2024/ctf_logo.jpg"]
+++

<!--more-->
{{< image src="/img/p3rf3ctr00t_ctf_2024/ctf_logo.jpg" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction
This weekend I had some fun participating in the P3RF3CTR00T CTF 2024 by [p3rf3ctr00t](https://twitter.com/p3rf3ctr00t) ctf team.

This is my second writeup, I'll cover the forensics category specifically the **Streams and Secrets** set. I am not really that into forensics but I had to give it a try to score some points, I also got first blood on some of them. I was going solo in this challenge under **NoPwnNoGain**.

## Part 1

We start of with the challenge info.

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_1.png" alt="" position="center" style="border-radius: 8px;" >}}

We are given 2 files, a script and the challenge artifact which we can download from the dump link.

This first one was easy to solve, I just got strings from the artifact.

```sh
strings -30 \$MFT.copy0
```
The only user I saw in the output that made sense was analyst so this was the flag.

Flag: `r00t{Analyst}`

## Part 2

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_4.png" alt="" position="center" style="border-radius: 8px;" >}}


Here I faced a bit of a challenge since I couldn't work with this file on linux anymore. Based on the strings, it was clear I needed to use Windows. 

I also did a lot of googling around what MFT. I found this resource [https://aboutdfir.com/toolsandartifacts/windows/mft-explorer-mftecmd/](https://aboutdfir.com/toolsandartifacts/windows/mft-explorer-mftecmd/) that guided me on the using MFTECmd and MFT Explorer.

```ps
 .\MFTECmd.exe -f '.\$MFT.copy0' --csv .
 ```
{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_2.png" alt="" position="center" style="border-radius: 8px;" >}}

With the csv extracted, we can through the information. I applied a filter on the filename to only show filenames with the name secret.

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_3.png" alt="" position="center" style="border-radius: 8px;" >}}

We can see we have two last modified columns, I used the most latest of the two.

Flag: `r00t{2024-10-07_21:52:47}`

## Part 3

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_5.png" alt="" position="center" style="border-radius: 8px;" >}}

This was only solvable after getting the original contents of the secret.txt file. So we needed the encryption key as well as the contents of the file. From the awesomeScript.py we know the encryption function.

Using MFTECmd, we need to get the EntryNumber and SequenceNumber to dump the details.

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_6.png" alt="" position="center" style="border-radius: 8px;" >}}


```ps
.\MFTECmd.exe -f '.\$MFT.copy0' --de 107744-3
```

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_7.png" alt="" position="center" style="border-radius: 8px;" >}}

We now have the contents of secrets.txt and the encryption_key that was stored in the alternate data stream. This was already indicated in the csv output.

MFT Explorer could have also been used, it gives a nice GUI experience but is quite slow. This is also something highlighted in the resource I found, so be patient when you first load the MFT file into MFT Explorer.

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_8.png" alt="" position="center" style="border-radius: 8px;" >}}

MFT Explorer can give us all the data at one glance.

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_9.png" alt="" position="center" style="border-radius: 8px;" >}}


With the key we can craft our decode script as below and get the original contents of secrets.txt

```py
from cryptography.fernet import Fernet
import base64
key = 'MVJhfcwOV33RxMzyF1H6J9X5IVbyfzHbVHMqXP6HN7Q=' 

message = 'gAAAAABnBFRI3Z3tfxy7hD4tfW_8Lkd4hwFOXxGkguaty3Z2zTzehVjBZhs9Q57y8g--0rTvkaZw44o-Nc0NxLFHqEYPiLab0FYXf7Y-34Rz27tKq_IFClITfXafCFR5BQb07PawxhP-'

f = Fernet(key)
encrypted_message = f.decrypt(message.encode())
print(str(encrypted_message,"utf-8"))
```
Once decoded, we get the following.

```txt
secret={M4st3r_F1l3_t4bl3_1n_ntfs}
```

Since it's a txt file, the size will be equal to the size of the text which in this case is 34.

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_10.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: `r00t{34}`

Based on the solves of the next challenges, seems the decryption was the biggest hurdle.

## Part 4

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_11.png" alt="" position="center" style="border-radius: 8px;" >}}

We already have the encryption key from the previous step.

Flag: `r00t{MVJhfcwOV33RxMzyF1H6J9X5IVbyfzHbVHMqXP6HN7Q=}`

## Part 5

{{< image src="/img/p3rf3ctr00t_ctf_2024/streams_and_secrets_12.png" alt="" position="center" style="border-radius: 8px;" >}}


Flag: `r00t{M4st3r_F1l3_t4bl3_1n_ntfs}`

## Conclusion

I definitely learnt a thing or two here about forensics on the Master File Table (MFT). Cheers to the author [boynamedboy](https://x.com/festusgichohi1)