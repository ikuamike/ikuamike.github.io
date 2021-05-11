+++
title = "RaziCTF 2020 WriteUp: Chasing a lock"
date = "2020-10-28"
author = ""
cover = ""
tags = ["CTF", "Android", "jadx-gui", "frida"]
keywords = ["", ""]
description = ""
showFullContent = false
+++
<!--more-->

# Summary

I recently participated in [RaziCTF 2020](https://ctftime.org/event/1167) with team 
[fr344aks](https://ctftime.org/team/112710) and I was able to solve an android challenge that I thought
needs a proper writeup. 
I was able to reverse engineer the provided app and use frida for dynamic analysis for a quick win.

### Challenge Description:

{{< image src="/img/razictf_2020/screenshot1.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

I first installed the app in my genymotion emulator to see the basic functionality. There is a padlock icon 
constantly changing position on the screen and text at the bottom that says ```20000 to break the lock```.

{{< image src="/img/razictf_2020/screenshot2.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

When I click on the padlock the number decreases by 1. This is definitely chasing a lock as the challenge name.

The goal seems pretty straightforward, we need to get the value to zero to get the flag. Now to decompile the app and see if we
can find a way. Like my previous writeup, I'll use jadx-gui.

From the AndroidManifest.xml we get the MainActivity that is loaded when the app starts.

{{< image src="/img/razictf_2020/screenshot3.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

Looking at the MainActivity code we see a switcher object being initialized just before the if statement that does some
check before displaying the flag. The switcher object has a method ```run``` that takes in a value ```i``` and the output is checked 
if it's not null and then the flag is displayed.

{{< image src="/img/razictf_2020/screenshot4.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

Let's look at run method in the switcher class as it's output determines whether the flag is displayed.
In the switcher code we can see the method ```run```. Instead of trying to figure out what each line does, I saw it easier to just hook into 
this method and manipulate the ```i``` value, which at this point I believe is the 20000 value.

{{< image src="/img/razictf_2020/screenshot5.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

I found two ways to solve this with frida, the run method was actually the one returning the flag:

First one is to hook into the method and change the input value ```i``` as it gets called.
```js
Java.perform(function(){
  var switcher = Java.use("com.example.razictf_2.switcher");
  var run = switcher.run;
  console.log("")
  console.log("Hooking...");
  run.overload('int').implementation = function (x){
    console.log("[*] Original Input: " + x);
    var ret = this.run(x)
    console.log("[*] Original return: " + ret)
    console.log("[*] Modified Input: 0");
    console.log("[*] Modified return: " + this.run(0))
    return ret;
 };
});
```
When I click on the padlock in the app, the flag is given.

{{< image src="/img/razictf_2020/screenshot6.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

Second one is to just initialize the switcher object ourselves and call the run method directly with our input without hooking.
This way I don't need to interact with the app.

```js
Java.perform(function(){
  var switcher = Java.use("com.example.razictf_2.switcher");
  var switcherInstance = switcher.$new();
  console.log("");
  console.log("Flag: " + switcherInstance.run(0));
});
```

{{< image src="/img/razictf_2020/screenshot7.png" alt="razictf" position="center" style="border-radius: 8px;" >}}

```Flag:  RaziCTF{IN_HATE_0F_RUNN!NG_L0CK5}```

# Conclusion
This was a good learning experience for me especially the second part which is quicker and has less code.

You can reach/follow me on Twitter if you have some feedback or questions.

Twitter: [ikuamike](https://twitter.com/ikuamike)

Github: [ikuamike](https://github.com/ikuamike)
