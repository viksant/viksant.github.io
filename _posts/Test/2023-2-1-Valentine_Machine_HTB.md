---
title: Valentine Machine HTB
date: 2023-02-1 00:18:00 +0700
categories: [TOP_CATEGORIE, SUB_CATEGORIE]
tags: [machines]
---


## Enumeration

```java
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
| 1024 964c51423cba2249204d3eec90ccfd0e (DSA)
| 2048 46bf1fcc924f1da042b3d216a8583133 (RSA)
|_ 256 e62b2519cb7e54cb0ab9ac1698c67da9 (ECDSA)
80/tcp open http Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.2.22 (Ubuntu)
443/tcp open ssl/http Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_ssl-date: 2023-02-04T16:57:58+00:00; +2s from scanner time.
| ssl-cert: Subject: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
|_Not valid before: 2018-02-06T00:45:25
|_Not valid after: 2019-02-06T00:45:25
|_http-server-header: Apache/2.2.22 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: 1s
```

We see a subdomain named `valentine.htb`.

Applying a directory fuzzing, we find `/dev` which contains 2 files: `hype_key and notes.txt`. We will also apply a file fuzzing, for which we will find:
```shell
/index.php (Status: 200) [Size: 38] ```index.php (Status: 200) [Size: 38]
/index (Status: 200) [Size: 38]
/.html (Status: 403) [Size: 286]
/dev (Status: 301) [Size: 312] [--> http://valentine.htb/dev/]
/encode.php (Status: 200) [Size: 554]
/encode (Status: 200) [Size: 554]
/decode (Status: 200) [Size: 552]
/decode.php (Status: 200) [Size: 552]
/omg (Status: 200) [Size: 153356]
/.html (Status: 403) [Size: 286]
```


We see that the hype_key file has a code encoded in hexadecimal format. If we use it to decode it, we see that it is an RSA class that surely will serve us to connect through SSH. Now it is time to find out which user it is for. 

## User Exploit

Searching the internet, we see an nmap script called heartbleed, which is a critical vulnerability in the OpenSSL encryption library, used to protect sensitive information on many websites and servers. The vulnerability allows an attacker to read system memory, which can include sensitive information such as passwords, banking information, cryptographic keys and other private data.

To do this, we will download two fundamental nmap scripts:

```shell
wget https://nmap.org/nsedoc/scripts/ssl-heartbleed.html
wget https://svn.nmap.org/nmap/nselib/tls.lua
```

And finally we execute it:
```shell
nmap -sV -p 443 --script=ssl-heartbleed valentine.htb
```

And it reports the following:

```java
PORT STATE SERVICE VERSION
443/tcp open ssl/http Apache httpd 2.2.22
|_http-server-header: Apache/2.2.22 (Ubuntu)
| ssl-heartbleed: 
| VULNERABLE:
| The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
| State: VULNERABLE
| Risk factor: High
| OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
| References:
| https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
| http://www.openssl.org/news/secadv_20140407.txt 
|_ http://cvedetails.com/cve/2014-0160/
Service Info: Host: 10.10.10.136
```

We found the following exploit https://github.com/sensepost/heartbleed-poc, so let's run it.

When we do so, it returns a file called out.txt that if we run it enough times, it returns this: `$text=aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==` so let's decode it. We get `heartbleedbelievethehype`, which is the key to decode the hype_key obtained earlier, so let's connect via ssh
```shell
chmod 600 hype_key
ssh -i hype_key hype@10.129.223.47
# In case you get the following error: sign_and_send_pubkey: no mutual signature supported, run the following command:
ssh -o 'PubkeyAcceptedKeyTypes +ssh-rsa' -i hype_key hype@10.129.223.47
```

And we will be able to see the user.txt flag

## Root Exploit

Doing a `uname -a` we see that we have a linux version 3.2.0, which is vulnerable to Kernel exploit. I am going to use one from firefart : https://github.com/FireFart/dirtycow/blob/master/dirty.c

To run it: 
* we go to /tmp inside the machine:
* create the file and save it
* compile it: `https://github.com/FireFart/dirtycow/blob/master/dirty.c`
* We execute it: `./exploit.c`.
* And now we can become the user firefart, which has root permissions: `your firefart`.
