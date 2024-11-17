+++
title = "HTB Business CTF 2024 WriteUp - Misc"
date = "2024-05-22T00:25:15+03:00"
author = ""
authorTwitter = "" #do not include @
tags = ["CTF", "Misc", "Web", "Unicode", "Python", "Git"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
toc = true
cover = "/img/htb_business_ctf_2024/ctf_logo.jpg"
+++



## Introduction
In this post, I'll be covering solutions to the Misc Challenges from the [HTB Business CTF 2024](https://ctf.hackthebox.com/event/details/htb-business-ctf-2024-the-vault-of-hope-1474).


### 1. Hidden Path
This challenge was rated **Easy**. We are provided with files to download, allowing us to read the app's source code.

{{< image src="/img/htb_business_ctf_2024/misc/hidden_path_1.png" alt="" position="center" style="border-radius: 8px;" >}}

On reading the code, we see that the app accepts user input on the ***/server_status*** endpoint. It takes in ***choice*** parameter and something else. 

Initially I didn't immediately pick this out but after looking closely, my code editor highlights that there is another parameter. Hovering my mouse over it revealed that it was the invisible unicode character U+3164. This character also appears to be also used in the array listing command choices that can be executed.

What this means is that, we can supply a command to this character then select it as our command choice using the index in the array .

I copied the unicode character from this [unicode-explorer](https://unicode-explorer.com/c/3164) link, then crafted my request as below in burp.ㅤ

{{< image src="/img/htb_business_ctf_2024/misc/hidden_path_2.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: **HTB{1nvi5IBl3_cH4r4cT3rS_n0t_sO_v1SIbL3_2b06cb9b519dd72e472ccdadda3917ce}**

### 2. Locked Away
This challenge was also rated **Easy**. Here we are also provided the code. For the challenge.

{{< image src="/img/htb_business_ctf_2024/misc/locked_away_1.png" alt="" position="center" style="border-radius: 8px;" >}}

As seen, our input will be checked against a blacklist then passed to exec command.

Since exec will execute any python code we supply, we can override the blacklist list by assigning it something different rendering it useless then read the flag by calling the openchest function. 

```py
blacklist = str(1),str(2),str(3)
```

We can also drop into a shell by import pty module.

{{< image src="/img/htb_business_ctf_2024/misc/locked_away_2.png" alt="" position="center" style="border-radius: 8px;" >}}

Flag: **HTB{bYp4sSeD_tH3_fIlT3r5?_aLw4Ys_b3_c4RefUL!_ab204eba259a305414b338855f2fd91d}**


### 3. Zephyr
This challenge was rated **Medium**. We are also provided with the challenge files.

{{< image src="/img/htb_business_ctf_2024/misc/zephyr_1.png" alt="" position="center" style="border-radius: 8px;" >}}

On the terminal, when we navigate to the folder, we note that this is a git repo with only two files. My terminal prompt also picks up the main branch.

Let's check the commit history on the main branch. 

```sh
git --no-pager log
```
{{< image src="/img/htb_business_ctf_2024/misc/zephyr_2.png" alt="" position="center" style="border-radius: 8px;" >}}

We only have 2 commits, lets view the modification using git diff.

```sh
git --no-pager diff 1501  ae4f
```
{{< image src="/img/htb_business_ctf_2024/misc/zephyr_4.png" alt="" position="center" style="border-radius: 8px;" >}}

We see that only the ***database.db*** file has been modified. Let's checkout the initial commit to view what sensitive info was removed.

{{< image src="/img/htb_business_ctf_2024/misc/zephyr_3.png" alt="" position="center" style="border-radius: 8px;" >}}

From this initial commit, we get a piece of the flag: **\_gOT_thE_DB\_**.

Now let's check if there are any other branches and switch to them.

{{< image src="/img/htb_business_ctf_2024/misc/zephyr_5.png" alt="" position="center" style="border-radius: 8px;" >}}

On the ***w4rri0r-changes*** branch, we see there is a new commit. We can use *git log* again but with *-p* option to view the changes.

```sh
git --no-pager log -p
```

{{< image src="/img/htb_business_ctf_2024/misc/zephyr_6.png" alt="" position="center" style="border-radius: 8px;" >}}

We now have another piece of the flag: **HTB{g0t_tH3_p4s5**.

Finally, after some time going through the files trying to get the final piece. I discovered file changes in a git repo can also be stored in stash.

```sh
git --no-pager stash show -p
```

{{< image src="/img/htb_business_ctf_2024/misc/zephyr_7.png" alt="" position="center" style="border-radius: 8px;" >}}

After checking the stash, we now have the final piece of the flag: **g0T_TH3_sT4sH}**.

Flag: **HTB{g0t_tH3_p4s5_gOT_thE_DB_g0T_TH3_sT4sH}**
