+++
title = "Grayhat Red Team Village CTF 2020 WriteUp: Tunneler"
date = "2020-11-01"
author = ""
tags = ["CTF", "SSH", "Tunneling"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
+++


## Summary
During Grayhat Conference the Red Team Village hosted a beginner/intermediate CTF. Our CTF team fr334aks decided
to participate as we enjoyed the previous CTF created by them during DEFCON. I tackled the Tunneler challenges
that were exactly the same as the previous CTF. So with the less pressure it was a nice opportunity to make a writeup for 
ssh tunneling techniques. 

I am writing this to serve as a personal reference for ssh tunneling as it has a very good practical aspect to use as an example. I will show different 
ways these challenges could have been solved.

## 1. Bastion

Challenge description:
```
Connect to the bastion host 104.131.101.182

User: tunneler 
Password: tunneler 
SSH Port: 2222
```

Solution:

```sh
ssh tunneler@104.131.101.182 -p 2222
```
Once we ssh in, we get the following welcome message and also the bastion host is connected to multiple networks. We don't see the ip we used to ssh in as 
we are droppped in a docker container.

{{< image src="/img/grayhat_2020/screenshot1.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 2. Browsing Websites

Challenge Description:
```
Browse to http://10.174.12.14/
```

This challenge requires a local port forward.

Solution 1:

Using ssh command.

```sh
ssh tunneler@104.131.101.182 -p 2222 -L 8000:10.174.12.14:80
```
Then use curl to reach the website.
```sh
curl 127.0.0.1:8000
```

Solution 2:

Using ssh config and add ```LocalForward``` entry.

```
Host bastion
 Hostname 104.131.101.182
 User tunneler
 Port 2222
 LocalForward 8000 10.174.12.14:80
```
Then ssh into bastion and run curl as above.

Solution 3:

Running sshuttle to add a route to the target ip, this utilizes the ssh config for bastion without the localforward entry.
```sh
sshuttle -r bastion 10.174.12.14/32
```
The beauty about this one is that I can reach the ip directly now.

```sh
curl 10.174.12.14
```

{{< image src="/img/grayhat_2020/screenshot2.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 3. SSH in tunnels

Challenge Description:
```
SSH through the bastion to the pivot.

IP: 10.218.176.199 
User: whistler 
Pass: cocktailparty
```
For this challenge we need to jump through the bastion host and access the pivot.

Solution 1:

Using the ```-J``` option in ssh command, this sets up a proxyjump.
```sh
ssh -J tunneler@104.131.101.182:2222 whistler@10.218.176.199
```
Solution 2:

Using ssh config, this builds on the config we started with in the previous challenge and uses ```ProxyJump```.
```
Host pivot1
 Hostname 10.218.176.199
 User whistler
 ProxyJump bastion
```
Then all we need to do is run the following command and see that pivot1 is also connected to more than one network.
```sh
ssh pivot1
```

{{< image src="/img/grayhat_2020/screenshot3.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 4. Beacons Everywhere

Challenge Description:
```
Something is Beaconing to the pivot on port 58671-58680 
to ip 10.112.3.199, can you tunnel it back?

NOTE: IT IS THE SAME ON EACH PORT ONLY USE ONE PORT 
AND REMOVE YOUR TUNNEL WHEN YOU ARE DONE
```
For this challenge we need to achieve a remote port forward.

Solution 1:

Using ssh command. First we proxyjump then perform the remote port forward.
```sh
ssh -J tunneler@104.131.101.182:2222 whistler@10.218.176.199 -R 10.112.3.199:58671:127.0.0.1:5000
```
Then on another terminal window, open a netcat listener.
```sh
nc -lvnp 5000
```
Solution 2:

Using ssh config. For this I'll add a ```RemoteForward``` option under pivot1.
```
Host pivot1
 Hostname 10.218.176.199
 User whistler
 ProxyJump bastion
 RemoteForward 58671 127.0.0.1:5000
```
Then we just run ```ssh pivot1``` and open a netcat listener.

{{< image src="/img/grayhat_2020/screenshot4.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 5. Beacons Annoying

Challenge Description:
```
Connect to ip: 10.112.3.88 port: 7000, a beacon awaits you
```
This challenge needs a local port forward and a remote port forward.

Solution:

Using ssh command, here we proxyjump and then forward localport 7000 to our target at port 7000.
```sh
ssh -J tunneler@104.131.101.182:2222 whistler@10.218.176.199 -L 7000:10.112.3.88:7000
```
Then connect using netcat. 
```sh
nc -vn 127.0.0.1 7000
```

{{< image src="/img/grayhat_2020/screenshot5.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

Next step is using a reverse port forward to receive the flag, we need to be quick to set the port as the flag is sent after 15s.

```sh
ssh -J tunneler@104.131.101.182:2222 whistler@10.218.176.199 -R 10.112.3.199:17172:127.0.0.1:6000
```
Then on another terminal open a listener on port 6000.
```sh
nc -vn 127.0.0.1 6000
```

{{< image src="/img/grayhat_2020/screenshot6.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}


It was an annoying beacon indeed, we needed 4 terminal panes open (I use tmux).

## 6. Scan me

Challenge Description:
```
We deployed a ftp server but we forgot which port, find it and connect

ftp: 10.112.3.207 user: bishop pass: geese
```

For this challenge we need to perform a portscan then a local portforward. However, port scanning over a tunnel can be quite slow
or sometimes fail. I ended up uploading an nmap binary to pivot1 and then scan the target.

Solution:

Building on the previous ssh config, I'll use scp to upload nmap. Nmap needs a services file to run, we need to upload that as well. Then from pivot1
we can perform a full portscan.
```sh
scp /opt/nmap-7.91SVN-x86_64-portable/nmap pivot1:/tmp
scp /opt/nmap-7.91SVN-x86_64-portable/data/nmap-services pivot1:/tmp
./nmap --datadir /tmp -p- 10.112.3.207
```

{{< image src="/img/grayhat_2020/screenshot7.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

After getting the port, let's use sshuttle to add a route to the ftp server.
```sh
sshuttle -r pivot1 10.112.3.207/32
```

Next is to just connect to the ftp server, the flag was in the banner.

{{< image src="/img/grayhat_2020/screenshot8.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 7. Another Pivot

Challenge Description:
```
Connect to the second pivot

IP: 10.112.3.12 
User: crease 
Pass: NoThatsaV
```
More proxyjumps.

Solution 1:

Multiple proxyjumps can be separated by a comma.
```sh
ssh -J tunneler@104.131.101.182:2222,whistler@10.218.176.199 crease@10.112.3.12
```
Solution 2:

I'll update ssh config with this new pivot.
```
Host pivot2
 Hostname 10.112.3.12
 User crease
 ProxyJump pivot1
```
Then run

```sh
ssh pivot2
```
This host is connected to multiple networks and it seems portforwarding is disabled here, therefore we'll have some fun with socat!

{{< image src="/img/grayhat_2020/screenshot9.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 8. SNMP

Challenge Description:
```
There is a snmp server at 10.24.13.161
```

This challenge is interesting because snmp runs over udp, you can't really create a udp tunnel. We'll tunnel udp inside a tcp tunnel.
Since in pivot2 tunneling is disabled we'll use socat.

Solution:

First we ssh into pivot2 then open a tcp listener which redirects traffic to udp out to snmp.
```sh
socat TCP4-LISTEN:5000,fork UDP4:10.24.13.161:161
```
Next we open a local portforward through pivot1 to the tcp port on pivot2 we opened above, I'll use the full ssh command here.

```sh
ssh -J tunneler@104.131.101.182:2222 whistler@10.218.176.199 -L 5000:10.112.3.12:5000
```
Then on our local machine we use socat again to redirect from local udp 161 to local port 5000 that would be forwarded by the above tunnel.
```sh
sudo socat UDP4-LISTEN:161,fork TCP:localhost:5000
```
Then finally we run snmpwalk against localhost and the traffic is forwarded through the long tunnel created.
```sh
snmpwalk -v1 -c public localhost
```

{{< image src="/img/grayhat_2020/screenshot10.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 9. Samba

Challenge Description:
```
There is a samba server at 10.24.13.10, find a flag sitting in the root file system /

nothing to find in the shares
```
Similar to the snmp one above we'll use socat to tunnel to the samba server.

Solution:

First we ssh into pivot2 and use socat to redirect traffic
```sh
socat -v TCP-LISTEN:5000,fork TCP:10.24.13.10:445
```
Then perform a local portforward through pivot1 to pivot2
```sh
ssh -J tunneler@104.131.101.182:2222 whistler@10.218.176.199 -L 5000:10.112.3.12:5000
```
Based on the challenge description, we need to access the root file system and nothing is in the shares. Sounds like
we need to exploit it as there's no other way to access the root file system.

Solution:

Run nmap to find the samba version and check for exploit in exploitdb using searchsploit.
```sh
nmap -sC -sV -p 5000 127.0.0.1
searchsploit samba
```

{{< image src="/img/grayhat_2020/screenshot11.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

There are several metasploit exploits listed for samba therefore we'll use it.
The exploit that worked is ```exploit/linux/samba/is_known_pipename``` and we get the flag.

{{< image src="/img/grayhat_2020/screenshot12.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

## 10. Browsing website 2

Challenge Description:
```
Browse to http://2a02:6b8:b010:9010:1::86/
```

This challenge shows portforwarding from ipv4 to ipv6. We'll use socat too on this one.

Solution:

First we ssh into pivot1 then use socat to redirect traffic from ipv4 to ipv6.
```sh
socat -v TCP-LISTEN:5000,fork TCP6:[2a02:6b8:b010:9010:1::86]:80
```
Then forward local port 5000 through pivot1 to pivot 2
```sh
ssh pivot1 -L 5000:10.112.3.12:5000
```
Then make the web request to local port 5000 which will be forwarded through the tunnel created.
```sh
curl localhost:5000
```

{{< image src="/img/grayhat_2020/screenshot13.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

The final base ssh config looked like this without the port forwards:

```
Host bastion
 Hostname 104.131.101.182
 User tunneler
 Port 2222

Host pivot1
 Hostname 10.218.176.199
 User whistler
 ProxyJump bastion

Host pivot2
 Hostname 10.112.3.12
 User crease
 ProxyJump pivot1
```

## Conclusion

There are several techniques to make your tunneling much easier, for quick use like in a ctf I think crafting the full ssh command is better and
faster. For frequent connection, using sshuttle combined with a good ssh config can make your life much easier.

After all those connections here's a Network Diagram that visualizes the access we were able to achieve with the tunnels:

{{< image src="/img/grayhat_2020/Tunneler.png" alt="grayhat_redteam_ctf" position="center" style="border-radius: 8px;" >}}

Here's a nice writeup by [@Rayhan0x01](https://twitter.com/Rayhan0x01) on how he used a different approach using metasploit: 
```sh
https://rayhan0x01.github.io/ctf/2020/08/08/defcon-redteamvillage-ctf-tunneler-1,2,3,4,5,7,9.html
```
## Resources
Some resources I think are helpful in understanding ssh tunneling properly.
```sh
1. https://www.redhat.com/sysadmin/ssh-proxy-bastion-proxyjump
2. https://www.cyberciti.biz/faq/linux-unix-ssh-proxycommand-passing-through-one-host-gateway-server/
3. https://book.hacktricks.xyz/tunneling-and-port-forwarding
4. https://linuxize.com/post/how-to-setup-ssh-tunneling/
5. https://www.tecmint.com/create-ssh-tunneling-port-forwarding-in-linux/
6. https://app.cyberranges.com/scenario/5d5c06ed960f032f2eadd733
7. https://app.cyberranges.com/scenario/5d5eaf28960f032f2eae6add
8. https://app.cyberranges.com/scenario/5d640b0c960f032f2eb033a8
```
You can reach/follow me on Twitter if you have some feedback or questions.

Twitter: [ikuamike](https://twitter.com/ikuamike)

Github: [ikuamike](https://github.com/ikuamike)
