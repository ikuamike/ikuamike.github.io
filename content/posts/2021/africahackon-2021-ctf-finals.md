+++
title = "AfricaHackon 2021 CTF Finals WriteUp"
date = "2021-11-09"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["CTF"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
images = ["/img/ahfinals2021/ahctf1.jpg"]
+++

<!--more-->
{{< image src="/img/ahfinals2021/processing0.png" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction

Last week [@mystic_kev](https://twitter.com/mystic_kev) and I participated in the AfricaHackon 2021 CTF finals as team NoPwnNoGain, we managed position 4.
We were able to solve 3 challenges with first blood on 2 of them. This writeup demonstrates how I was able to solve one of them.

## Challenges

### 1. Processing 

Challenge description:

{{< image src="/img/ahfinals2021/processing.png" alt="" position="center" style="border-radius: 8px;" >}}

For this challenge we are provided a zip file with the java apps for the challenge.

{{< image src="/img/ahfinals2021/processing2.png" alt="" position="center" style="border-radius: 8px;" >}}

The Challenge file is a shell script that allows for executing the challenge easily. On executing it we get a gui app.

{{< image src="/img/ahfinals2021/processing3.png" alt="" position="center" style="border-radius: 8px;" >}}

When we supply the wrong password we get access denied.

{{< image src="/img/ahfinals2021/processing4.png" alt="" position="center" style="border-radius: 8px;" >}}

Since this are jar files we can decrypt them using jadx-gui. I'll focus on the Challenge.jar file as it's the entrypoint 
of the challenge. From jadx-gui we can spot the functions responsible for the input interface and for checking the password.

{{< image src="/img/ahfinals2021/processing5.png" alt="" position="center" style="border-radius: 8px;" >}}

The next logical step would be to reverse engineer the code and build the flag from there. If you would like to see how that
approach would have been done see these writeups.

[https://lvmalware.github.io/writeup/2021/11/06/Africahackon-Finals.html](https://lvmalware.github.io/writeup/2021/11/06/Africahackon-Finals.html)

[https://trevorsaudi.medium.com/africahackon-2021-ctf-finals-8111f8edc408](https://trevorsaudi.medium.com/africahackon-2021-ctf-finals-8111f8edc408)

I used a different approach that I learnt from another ctf that I would like to share.

By using a tool called [recaf](https://github.com/Col-E/Recaf) we can edit the application's code and repackage it again to run
our modified code. This means that we can utilize the existing functions and classes in the application to achieve our goal.

{{< image src="/img/ahfinals2021/processing6.png" alt="" position="center" style="border-radius: 8px;" >}}

```java
public boolean checkPW(String pw) {
	ArrayList<Character> output = new ArrayList<Character>();
	int i = 0;
	while (i < this.s_key.length) {
	    char test = PApplet.parseChar(Challenge.unhex(this.s_key[i]));
	    output.add(test);
	    ++i;
	}
	System.out.println(output);
	return true;

}
```
I edited the checkPW function as shown above, and then saved the new Challenge.jar by exporting it.

```
File -> Export Program
```

With our Challenge.jar modified we can run the challenge then supply any input as the password check is removed and get the flag. I am not so good with java 
so I let the output appear as an array then cleaned it up manually.

{{< image src="/img/ahfinals2021/processing7.png" alt="" position="center" style="border-radius: 8px;" >}}

```sh
flag{Processing_1s_a_c00l_Pr0gramm1ng_language}
```

### 2. Phished

I was to solve phished and get **first blood** but I followed John Hammond's video on a previous ctf that solved this similar challenge so I'll just link it
as the explanation is good enough. Only difference is I used [cyberchef](https://gchq.github.io/CyberChef/) for find and replace to clean up the output 
from the xls instead of an editor.

[https://youtu.be/QgHhgwKL79Q](https://youtu.be/QgHhgwKL79Q)


[@trevorsaudi](https://twitter.com/trevorsaudi) also shows how to solve this in his writeup.

```
https://trevorsaudi.medium.com/africahackon-2021-ctf-finals-8111f8edc408
```

## Conclusion

Thanks to [@AfricaHackon](https://twitter.com/AfricaHackon) and the partners for the CTF, we don't get to play many locally organised ctfs so this was good to compete against local teams. 

You can still play the challenges by visiting this link [https://ctfroom.com/vaults/](https://ctfroom.com/vaults/)

